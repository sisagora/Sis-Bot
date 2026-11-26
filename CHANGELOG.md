# Changelog

## 1.5.9 — 25/11/2026
- **Scan com piso de CU:** se a simulação do ScanForSurveyDataUnits der menos de 100 mil CU, o limite pedido é no mínimo 200 mil. Em 25/11 17:46 a simulação deu 29.659 CU, o limite saiu 69.659 e o scan falhou por "CU esgotado".
- `@namespace` passou de `https://sis-agora.realstudio.com.br/` para `https://sis-agora.com`.
- `@require` das bibliotecas aponta para este repositório (`lib/`), com os mesmos sha256 (antes: ImGroovin/SAGE-Lab-Assistant).
- RPC padrão do script: somente o RPC oficial do Star Atlas (os do dashboard e os do usuário continuam valendo por cima). Texto do painel corrigido (citava Ironforge/Helius).
- A partir desta versão o arquivo do script é a própria fonte (a pasta `fonte/` do projeto local ficou desatualizada desde a 1.5.1).

Versões anteriores (1.0.0–1.5.8) ficam só no projeto local.
