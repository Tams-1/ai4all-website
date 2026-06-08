# AI4ALL Platform — Website

Public website for AI4ALL Platform.

Live domains:
- www.ai4allplatform.com
- www.ai4allplatform.com.br

## Structure

- index.html — main landing page
- assets/ — logos and images
- clientes/ — per-client pages for Meta/WhatsApp compliance
- clientes/grupo-fani/ — Grupo Fani institutional page, privacy policy and terms

## Adding a new client

Copy clientes/grupo-fani/ as a template. Update CNPJ, address, contact and company name throughout all 3 files.

## Deploy

Served via nginx on Hostinger VPS (177.7.57.95) behind Traefik.
Domain: ai4allplatform.com → container ai4all-website on port 80.
