# SISTEMA DE RESERVAS DE HOTEL POR WELLINGTON D. ANGELO
Projeto individial da disciplina ES0008 -Programação Orientada a Objetos ministrada no semestre 2026.2

 ---
## Objetivo
O projeto consiste em desenvolver uma API de um sistema de reservas de hotel. Este permitirá acesso aos dados de hóspedes, quartos e reservas com check-in/check-out. Também devendo conter tratamentos para política de cancelamento, tarifas por temporada, bloqueios por manutenção e relatórios de desempenho.

---

## 🧱 Decisões técnicas

| Decisão | Escolha | Justificativa |
|---|---|---|
| Interface | CLI + API mínima (FastAPI) | As duas compartilham o mesmo `services/`. A CLI é a entrega principal exigida; a API serve como exercício de design de contratos e preparação para práticas de DevOps |
| Persistência | SQLite (`sqlite3`) | Arquivo único, sem dependência externa, transactions nativas, evolução natural para banco cliente-servidor. JSON foi descartado pela fragilidade em escritas concorrentes |
| Tarifação | Funções puras | Multiplicadores de temporada e fim de semana são calculados por diária, sem estado nem I/O — trivial de testar |
| Estados da reserva | Máquina de estados explícita | Transições `PENDENTE → CONFIRMADA → CHECKIN → CHECKOUT` (e `→ CANCELADA` / `→ NO_SHOW`) validadas em métodos de domínio; estado nunca é alterado por atribuição direta |
| ADR | Receita de **hospedagem** (diárias) / noites vendidas | Adicionais (frigobar, estacionamento) **não** entram no ADR, mas entram na receita total e no RevPAR. Decisão documentada e coberta por teste |
| Herança múltipla | Mixin `AuditavelMixin` (`criado_em`, `atualizado_em`) | Uso idiomático de herança múltipla para comportamento transversal, sem criar hierarquias de negócio artificiais |

## 🗂️ Estrutura

```
hotel/
├── models/            # POO puro: sem I/O, sem frameworks
│   ├── pessoa.py          # Pessoa (ABC) → Hospede
│   ├── quarto.py          # Quarto → QuartoSimples | QuartoDuplo | QuartoLuxo
│   ├── reserva.py         # Reserva agrega Pagamento e Adicional (composição)
│   ├── pagamento.py
│   ├── adicional.py
│   └── mixins.py          # AuditavelMixin (herança múltipla)
├── services/
│   ├── tarifacao.py       # funções puras: temporada + fim de semana
│   ├── reserva_service.py # disponibilidade, check-in/out, cancelamento, no-show
│   ├── relatorios.py      # ocupação, ADR, RevPAR, receita por tipo
│   └── bloqueio.py        # manutenção de quartos
├── dados.py             # save/load SQLite (tabelas e migrations)
├── settings.py          # carrega settings.json
├── seed.py              # 6–8 quartos + 3 períodos de temporada
├── cli.py               # CLI com subcomandos (argparse/typer)
├── api/                 # API FastAPI (bônus, compartilha services/)
│   ├── main.py
│   ├── schemas.py
│   └── routers/
└── __main__.py          # python -m hotel ...
tests/                   # ≥ 15 casos (pytest)
config/settings.json
```

## 🔁 Diagrama de classes (resumo)

```
                 AuditavelMixin
                 (criado_em, atualizado_em)
                       △        △
        Pessoa (ABC)   │        │
            △          │        │
        Hospede ───────┴────────┤
            │ 1                 │
            │                   │
            │ *                 │
        Reserva ────────────────┘        Quarto
        │ 1   │ 1        │ *                  △
        │     │          │                 ┌──┴──┐
   Pagamento  Adicional  │        QuartoSimples | Duplo | Luxo
        (* composição)   │ *              │
                         └───────────────┘
```

- **Herança**: `Quarto` com subclasses por tipo (tarifa/capacidade sobrescritas); `Pessoa` abstrata → `Hospede`; `AuditavelMixin` (múltipla).
- **Composição**: `Reserva` agrega `Pagamento` e `Adicional` (não existem sem a reserva).
- **Métodos especiais**: `Quarto.__str__/__repr__/__lt__`, `Reserva.__len__` (nº de diárias), `Reserva.__eq__/__hash__` (mesmo quarto + intervalo).
- **Encapsulamento**: `capacidade ≥ 1`, `tarifa_base > 0`, `entrada < saída`, `num_hospedes ≤ capacidade` via `@property` com validação.

## 🚀 Como executar

### Pré-requisitos

- Python ≥ 3.11
- (Opcional) Docker / Docker Compose

### Instalação

```bash
git clone https://github.com/<usuario>/hotel-reservas.git
cd hotel-reservas
python -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"
```

### Seed (dados iniciais)

```bash
python -m hotel seed
# Cria 8 quartos (2 de cada tipo + extras) e 3 períodos de temporada
# em data/hotel.db, a partir de config/settings.json
```

### CLI

```bash
python -m hotel cadastrar-quarto --numero 101 --tipo DUPLO --capacidade 2 --tarifa 220
python -m hotel bloquear-quarto 101 --motivo "Vazamento" --inicio 2026-10-01 --fim 2026-10-03
python -m hotel reservar --hospede 1 --quarto 101 --entrada 2026-10-05 --saida 2026-10-08 --hospedes 2 --origem site
python -m hotel checkin 3
python -m hotel adicional 3 --descricao "Frigobar" --valor 35.50
python -m hotel pagar 3 --forma PIX --valor 780.00
python -m hotel checkout 3
python -m hotel relatorio ocupacao --inicio 2026-10-01 --fim 2026-10-31
python -m hotel relatorio revpar --inicio 2026-10-01 --fim 2026-10-31
```

### API (opcional)

```bash
uvicorn hotel.api.main:app --reload
# Documentação interativa: http://127.0.0.1:8000/docs
```

### Docker

```bash
docker compose up --build
# Persistência do SQLite via volume: data/hotel.db
```

### Configuração (`config/settings.json`)

```json
{
  "checkin_hora": "14:00",
  "checkout_hora": "12:00",
  "temporadas": [
    {"nome": "Alta", "inicio": "12-20", "fim": "01-05", "multiplicador": 1.3},
    {"nome": "Feriados", "inicio": "07-01", "fim": "07-31", "multiplicador": 1.2}
  ],
  "fds_multiplicador": 1.1,
  "cancelamento": {"janela_horas": 48, "multa_pct": 0.5},
  "no_show_tolerancia_min": 120
}
```

## 🧪 Testes

```bash
pytest -v
# ou com cobertura
pytest --cov=hotel --cov-report=term-missing
```

Suíte com **15+ casos** cobrindo os cenários exigidos:

- `test_disponibilidade.py` — sobreposição de reservas, bloqueio por manutenção
- `test_capacidade.py` — hóspedes > capacidade do quarto
- `test_tarifacao.py` — temporada, fim de semana e combinação dos dois
- `test_estados.py` — transições válidas e inválidas da reserva
- `test_cancelamento.py` — multa dentro/fora da janela de 48h
- `test_noshow.py` — tolerância e liberação do quarto
- `test_relatorios.py` — ADR, RevPAR e taxa de ocupação sobre dados conhecidos

## 🔄 CI/CD

Pipeline em GitHub Actions a cada push/PR:

1. **lint** — `ruff check .`
2. **testes** — `pytest` (matriz Python 3.11/3.12, banco em memória)
3. **segurança** — `pip-audit` (vulnerabilidades de dependências)

```bash
# executar localmente
ruff check . && pytest
```

## 🗺️ Roadmap (entregas semanais)

| Semana | Entrega |
|---|---|
| 1 | UML textual, scaffold com classes vazias + docstrings, README inicial |
| 2 | Classes base com `@property` e métodos especiais, testes unitários |
| 3 | Composição (pagamentos/adicionais), persistência SQLite, relatório de ocupação |
| 4 | Check-in/out, cancelamento, no-show, tarifação, CLI funcional |
| 5 | ADR/RevPAR, README final, suíte de testes completa, **tag v1.0** |

## 📌 Tags de entrega

As entregas semanais são marcadas com tags no repositório:

```bash
git tag -a semana-1 -m "UML + scaffold"
git push origin semana-1
```

## 📝 Convenções

- Commits no estilo [Conventional Commits](https://www.conventionalcommits.org/) (`feat:`, `fix:`, `test:`, `refactor:`)
- Nenhum valor hardcoded de negócio: tudo passa por `settings.json` ou variável de ambiente (`DATABASE_URL`)
- Regras de negócio **nunca** em `cli.py` ou `api/` — somente em `models/` e `services/`

---
Projeto acadêmico — Bacharelado em Engenharia de Software, UFCA.

   ```

4. **Popular banco de dados inicial (Seed):**
   ```bash
   python seed.py
   ```

5. **Executar a API (FastAPI):**
   ```bash
   uvicorn main:app --reload
   ```
   Acesse a documentação interativa Swagger em: [http://127.0.0.1:8000/docs](http://127.0.0.1:8000/docs)

---

## Execução dos Testes

O projeto utiliza `pytest` para a suíte de testes unitários e de integração, garantindo o funcionamento das regras de negócio e validações.

```bash
# Executar todos os testes
pytest

# Executar com relatório detalhado
pytest -v
```

---

## 📈 Relatórios de Desempenho Incluídos

A API disponibiliza endpoints/consultas para os seguintes indicadores da indústria hoteleira:
- **Taxa de Ocupação (%)**: Percentual do total de quartos ocupados em um determinado período.
- **ADR (Average Daily Rate)**: Receita total de hospedagem dividida pelo número de diárias efetivamente vendidas.
- **RevPAR (Revenue per Available Room)**: Receita total dividida pelo total de quartos disponíveis no período.
- **Cancelamentos e No-shows**: Relatório quantitativo e financeiro de perdas e multas aplicadas.

---
