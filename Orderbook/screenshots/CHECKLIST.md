# Screenshot / Visual Checklist — AB Crypto (Orderbook)

Capture these yourself (I'm not taking screenshots). File naming suggestion in brackets.

## App screens

1. **Trading view / order book UI** [`01-trading-view.png`] — the main frontend screen
   showing `TradingChart.jsx` plus the live order book (bids/asks) for an instrument. This is
   the single most important screenshot — it's the visible payoff of the whole matching-engine
   backend.
2. **Order placement flow** [`02-place-order.png`] — the form/modal for submitting a buy or
   sell order.
3. **Trade history** [`03-trade-history.png`] — the executed-trades view (trade-service data
   surfaced in the UI).
4. **Login/auth screen** [`04-auth.png`] — quick proof the auth-service/JWT flow is wired up
   end to end, doesn't need to be elaborate.

## Infra/tooling screenshots (these sell the "real system," not just the UI)

5. **Kafdrop topic view** [`05-kafdrop-topics.png`] — `localhost:9000` while the system is
   running, showing the `orders-created`, `orders-cancelled`, `trades-executed` topics with
   live message traffic. This is strong, concrete evidence the event-driven architecture
   actually works, not just described in the README.
6. **Docker Compose services running** [`06-docker-ps.png`] — terminal output of `docker
   compose ps` (or Docker Desktop) showing all services (gateway, auth, engine, orders,
   trades, postgres, kafka, kafdrop, frontend) up and healthy.
7. **(If you still have access) Jenkins pipeline run** [`07-jenkins-pipeline.png`] — a
   successful pipeline showing the build → ECR push → deploy stages, if you can still access
   the Jenkins instance. Skip if it's no longer running — don't spin up infrastructure just
   for this screenshot.

## Diagrams to sketch (not screenshots)

8. **Event flow diagram** [`08-event-flow.png`] — `Order placed (order-service) →
   orders-created topic → Matching Engine consumes → matches against OrderBook → 
   trades-executed topic → Trade Service persists + Order Service updates order status`. The
   README already has a text version of this; a cleaner visual version is worth having for a
   portfolio page.
9. **(Optional) order book data structure sketch** — a small diagram of the
   `ConcurrentSkipListMap` (price levels, sorted) with a FIFO queue at each level (time
   priority) — good for a technical-deep-dive blog post or portfolio detail page, since that's
   the most "computer science-y" part of the project.

## Notes

- Use fake/test instrument symbols and order sizes — don't worry about this one, there's no
  real money or personal data involved, but keep it looking like plausible demo data rather
  than `test123`/`asdf` placeholders.
- If the AWS-hosted deployment isn't currently running, it's fine to capture everything from
  `docker-compose.dev.yml` locally — the architecture is identical, just without the Jenkins
  gap.
