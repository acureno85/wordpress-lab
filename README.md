# WordPress Support Lab

Personal WordPress environment for learning support troubleshooting.
Built with Docker Compose (WordPress + MySQL + phpMyAdmin).

## Environment
- WordPress 6.x (latest)
- MySQL 8.0
- Plugins: WooCommerce, Yoast SEO, Contact Form 7, Wordfence, WP Super Cache

## Documented Issues & Resolutions
- [Ticket #001 — WooCommerce blank checkout](docs/support-ticket-001.md)
- [Ticket #002 — Memory limit errors](docs/support-ticket-002.md)
- [Ticket #003 — Broken permalinks after migration](docs/support-ticket-003.md)
- [Ticket #004 — White Screen of Death diagnosis](docs/support-ticket-004.md)
- [Ticket #005 — Plugin conflict resolution](docs/support-ticket-005.md)

## Setup
\`\`\`bash
docker compose up -d
# Visit http://localhost:8080
\`\`\`
