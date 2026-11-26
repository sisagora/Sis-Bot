# Sis-Bot

Script Tampermonkey do Sis-Ágora para o SAGE Labs (Star Atlas). Motor: SLY Assistant 0.7.35 (SLY w/ Contributions by niofox, SkyLove512, anthonyra, [AEP] Valkynen, Risingson, Swift42), configurado pelo dashboard do Sis-Ágora com aceite local.

## Estrutura
- `Sis-Bot.user.js` — a versão atual (igual à última de `versoes/`).
- `versoes/Sis-Bot-X.Y.Z.txt` — uma cópia de cada versão publicada, a partir da 1.5.9.
- `lib/` — bibliotecas que o script carrega com `@require` (cópias nossas, com sha256 conferido):
  - `anchor-browserified.js` — `f29ef75915bcf59221279f809eefc55074dbebf94cf16c968e783558e7ae3f0a`
  - `buffer-browserified.js` — `4fa88e735f9f1fdbff85f4f92520e8874f2fec4e882b15633fad28a200693392`
  - `bs58-browserified.js` — `87095371ec192e5a0e50c6576f327eb02532a7c29f1ed86700a2f8fb5018d947`

As três vieram do repositório ImGroovin/SAGE-Lab-Assistant e têm exatamente os mesmos hashes que o script já usava; hoje não dependemos mais da conta de terceiros.

## Instalar / atualizar
**Atualizar quem já tem o Sis-Bot:** abra o script no editor do Tampermonkey e cole o código novo por cima. Não instale por link: o `@namespace` mudou na 1.5.9 e o Tampermonkey trataria como outro script, com o armazenamento (hot wallet, configurações) vazio.

## Regras de versão
- Nova versão = novo arquivo em `versoes/`, `Sis-Bot.user.js` atualizado, linha no `CHANGELOG.md` e uma tag `vX.Y.Z`.
- Não publicar chaves (hot wallet, tokens, chaves de RPC privadas) neste repositório.
