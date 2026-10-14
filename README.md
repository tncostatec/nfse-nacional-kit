# NFS-e Nacional — kit de emissão em Python

[![CI](https://github.com/tncostatec/nfse-nacional-kit/actions/workflows/ci.yml/badge.svg)](https://github.com/tncostatec/nfse-nacional-kit/actions/workflows/ci.yml) ![Python 3.11+](https://img.shields.io/badge/python-3.11%2B-blue) [![Licença MIT](https://img.shields.io/badge/licen%C3%A7a-MIT-green)](LICENSE)

Emissão, consulta e cancelamento de **NFS-e no padrão Nacional** (SEFIN / gov.br),
extraído de um sistema em produção que emite notas de verdade desde julho/2026.

Não é uma biblioteca genérica de "notas fiscais": é o caminho específico do
**Sistema Nacional NFS-e** — o padrão unificado que substitui os mil sistemas
municipais diferentes. Se o seu município já aderiu ao Nacional, isto emite. Se
ele ainda usa sistema próprio (Ginfes, ISSNet, WebISS...), isto **não** serve.

*English: issue, query and cancel Brazilian service invoices (NFS-e) on the national standard
(SEFIN / gov.br) from Python: DPS XML builder, XMLDSig signing with an ICP-Brasil A1 certificate,
mTLS client, tolerant response parser and a catalogue of the validator pitfalls that are not in the
official manual. Three dependencies, no framework, no database. Documentation is in Portuguese.*

---

## Comece por aqui (3 minutos)

```bash
python -m venv .venv
# Windows:      .venv\Scripts\activate
# Linux/macOS:  source .venv/bin/activate

pip install -r requirements.txt
python exemplos/00_teste_local.py
```

O `00_teste_local.py` **não precisa de certificado nem de internet**. Ele gera um
certificado de teste, monta uma DPS, assina e confere a assinatura do mesmo jeito
que a SEFIN confere. Se ele imprimir `[OK] Tudo certo`, seu ambiente está pronto
e você já pode abrir o `dps_exemplo_assinada.xml` para ver o XML que sai daqui.

Depois: copie `.env.example` para `.env`, preencha, e rode
`python exemplos/01_emitir_nota.py`.

---

## O que tem aqui

```
nfse_core/          O núcleo. Sem banco, sem ORM, sem framework web.
  dps.py              monta o XML da DPS (a "nota" que você submete)
  signer.py           assina com certificado A1 (ICP-Brasil), padrão DF-e
  client.py           conversa com a SEFIN (mTLS + GZip + Base64)
  evento.py           monta o XML de cancelamento
  resposta.py         lê as respostas de forma tolerante a mudança de nome
  certificado.py      valida o A1: titular, CNPJ, validade, alerta de vencimento
  error_catalog.py    traduz os códigos de erro da SEFIN para português claro

exemplos/
  00_teste_local.py   valida seu ambiente — sem certificado, sem internet
  01_emitir_nota.py   emissão completa
  02_cancelar_nota.py cancelamento (evento e101101)
  03_consultar.py     consulta por chave, e por DPS (sua saída de emergência)

docs/
  ARMADILHAS.md       ⭐ LEIA. Os erros que custaram dias para descobrir.
  INTEGRACAO.md       Como plugar isto no seu sistema.
```

O fluxo inteiro de uma emissão:

```python
xml       = build_dps_xml(DpsData(...))          # monta
assinado  = sign_dps(xml, pfx_base64, senha)     # assina
bruta     = await SefinClient(ambiente, pfx_base64, senha).emitir_dps(assinado)
resultado = ler_resposta_emissao(bruta)          # lê (tolerante a mudança de nome)

if resultado.autorizada:
    guardar(resultado.chave_acesso, resultado.xml_nfse)   # o XML é o documento
else:
    mostrar(resultado.erros)                              # já traduzidos
```

O cancelamento é simétrico: `build_evento_cancelamento_xml` → `sign_evento` →
`registrar_evento` → `ler_resposta_evento`.

Tudo o que é do **seu** sistema — de onde vem o valor, quem é o tomador, onde a
emissão é gravada, quando ela dispara — fica fora do núcleo, de propósito.
Veja `docs/INTEGRACAO.md`.

---

## O que você precisa antes de emitir de verdade

1. **Certificado A1 (e-CNPJ) ICP-Brasil**, arquivo `.pfx` + senha. A1 é o de
   arquivo; A3 (token/cartão) **não** funciona aqui — a assinatura precisa da
   chave privada em memória.
2. **Inscrição Municipal** do prestador. Sem ela, erro `E0099`.
3. **Adesão do município ao Sistema Nacional.** Confira em
   <https://www.gov.br/nfse> antes de qualquer coisa. Município não aderido
   rejeita tudo com `E0084`/`E0099`, e nenhum ajuste de código resolve.
4. **Credenciamento na produção restrita** para testar. É um ambiente separado,
   com cadastro próprio.

---

## Avisos que valem tempo

- **Comece em `homologacao`.** Nota emitida em `producao` é documento fiscal
  real; cancelar depois envolve prazo municipal e justificativa.
- **O número da nota é sequencial e único por série.** Reenviar um número já
  usado é rejeitado. No seu sistema, isso precisa vir de um contador
  transacional no banco — nunca de um `max(numero) + 1` lido antes ## Licença e origem

Código extraído da camada fiscal de um ERP em produção (TN Costa Tecnologia), que emite notas de verdade
desde julho/2026. Contém só a camada fiscal: nenhuma lógica de negócio do sistema de origem.

Licença **MIT** (ver `LICENSE`): use, modifique e redistribua, mantendo o aviso de copyright.

Sem garantia: emissão de documento fiscal é responsabilidade de quem emite. Valide em produção
restrita e confira as regras do seu município antes de emitir a primeira nota real.

## Autor

**Tiago Costa** · TN Costa Tecnologia (Macapá, AP; atendimento em todo o Brasil)
[tncostatecnologia.com.br](https://www.tncostatecnologia.com.br/) · [LinkedIn](https://www.linkedin.com/in/tiago-costa-7470587a) · [GitHub](https://github.com/tncostatec)

Achou uma divergência nova do validador? Abra uma issue com o código de erro e o trecho do XML
(sem dados reais): a lista de armadilhas cresce assim.
