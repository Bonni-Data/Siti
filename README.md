# Siti"""Orchestratore della pipeline giornaliera.

Esempi:
  python main.py --mock                  # prova end-to-end senza rete
  python main.py                         # run reale (richiede GOOGLE_PLACES_API_KEY)
  python main.py --find-emails           # tenta estrazione email dai siti datati
  python main.py --send                  # invia le ROUTE A APPROVATE (gate multipli)
  python main.py --area "Rozzano" --settore "ristoranti"

Flusso: discovery -> dedup -> qualifica -> (email finder) -> classifica ->
analisi -> demo -> registro -> outreach (bozze) -> [invio gated] -> CSV + dashboard."""
from __future__ import annotations

import argparse

from studio35 import config, suppression, registry, storage
from studio35.discovery import PlacesClient
from studio35.qualify import qualify
from studio35.classify import classify
from studio35.analyze import analyze
from studio35.demo import generate_demo
from studio35.email_finder import find_email
from studio35.outreach import prepare, send_approved
from studio35.models import Route


def run(args) -> None:
    sectors = [args.settore] if args.settore else config.SETTORI
    areas = [args.area] if args.area else config.AREE

    print(f"[1/7] Discovery — {len(sectors)} settori x {len(areas)} aree "
          f"({'MOCK' if args.mock else 'Google Places'})")
    client = PlacesClient(mock=args.mock)
    leads = client.discover(sectors, areas, per_query=args.per_query)
    print(f"      {len(leads)} lead grezzi")

    # dedup: per place_id nuovo nella sessione e non gia' nel registro
    seen, known = set(), registry.known_place_ids()
    deduped = []
    for l in leads:
        if l.place_id and l.place_id not in seen and l.place_id not in known:
            seen.add(l.place_id)
            deduped.append(l)
    print(f"[2/7] Dedup — {len(deduped)} lead nuovi")

    qualified = []
    for l in deduped:
        qualify(l)
        if not l.qualified:
            continue
        # email finder opzionale solo per chi ha un dominio proprio (sito datato)
        if args.find_emails and l.website and not l.email:
            try:
                found = find_email(l.website)
                if found:
                    l.email = found
            except Exception:
                pass
        classify(l)
        if suppression.is_suppressed(l.email, l.phone, l.name):
            l.contact_status = "opt_out"
            qualified.append(l)
            continue
        analyze(l, use_llm=args.use_llm)
        generate_demo(l)
        registry.record(l)
        prepare(l)
        qualified.append(l)
        if len(qualified) >= args.target:
            break

    print(f"[3/7] Qualifica + analisi + demo — {len(qualified)} qualificati")
    routes = {r: sum(1 for l in qualified if l.route == r.value) for r in Route}
    print(f"[4/7] Routing — A:{routes[Route.A]}  B:{routes[Route.B]}  C:{routes[Route.C]}")
    print(f"[5/7] Bozze e note preparate in {config.DRAFT_DIR}")

    # invio gated (default OFF)
    if args.send:
        sent = 0
        for l in qualified:
            before = l.contact_status
            send_approved(l, do_send=True)
            if l.contact_status == "inviato" and before != "inviato":
                sent += 1
        print(f"[6/7] Invio — {sent} email inviate (solo ROUTE A approvate)")
    else:
        pending = sum(1 for l in qualified if l.contact_status == "bozza_in_revisione")
        print(f"[6/7] Invio disattivato — {pending} bozze ROUTE A in attesa di approvazione")

    csv_path = storage.write_csv(qualified)
    dash = storage.build_dashboard(deduped, qualified)
    storage.print_dashboard(dash)
    html_path = storage.write_dashboard_html(dash)
    print(f"[7/7] Output: {csv_path}\n        {html_path}")


def main():
    p = argparse.ArgumentParser(description="studio35 Business Development Agent")
    p.add_argument("--mock", action="store_true", help="usa dati fittizi, nessuna rete")
    p.add_argument("--find-emails", action="store_true", help="estrai email dai siti propri (datati)")
    p.add_argument("--use-llm", action="store_true", help="analisi via API Anthropic (richiede chiave)")
    p.add_argument("--send", action="store_true", help="invia le ROUTE A APPROVATE (gate multipli)")
    p.add_argument("--area", help="limita a una sola area")
    p.add_argument("--settore", help="limita a un solo settore")
    p.add_argument("--per-query", type=int, default=20, help="max risultati per query")
    p.add_argument("--target", type=int, default=config.DAILY_QUALIFIED_TARGET,
                   help="numero di lead qualificati da raggiungere")
    run(p.parse_args())


if __name__ == "__main__":
    main()
