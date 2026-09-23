# SISTEMA DE RESERVAS DE HOTEL POR WELLINGTON D. ANGELO
Projeto individial da disciplina ES0008 -Programação Orientada a Objetos ministrada no semestre 2026.2

 ---
## Objetivo
O projeto consiste em desenvolver uma API de um sistema de reservas de hotel. Este permitirá acesso aos dados de hóspedes, quartos e reservas com check-in/check-out. Também devendo conter tratamentos para política de cancelamento, tarifas por temporada, bloqueios por manutenção e relatórios de desempenho.

---
## Tecnologias

- Python 3.11+
- FastAPI (a API)
- Uvicorn (o servidor)
- SQLite (o banco de dados)
- Pytest (os testes)

---
## Como funciona

```
Cliente (navegador, curl, Postman)
        │  HTTP + JSON
        ▼
    api/  ──► services/ ──► models/ ──► SQLite
```

- `api/` só recebe a requisição, valida os dados e devolve a resposta
- `services/` tem as regras de negócio (disponibilidade, preço, multa)
- `models/` tem as classes (Quarto, Reserva, Hospede...)

**Regra do projeto:** nenhuma regra de negócio fica em `api/`.

---
## Estrutura

```
hotel/
├── models/         # Classes do domínio
├── services/       # Regras de negócio
├── api/
│   ├── main.py         # Cria o app e registra as rotas
│   ├── schemas.py      # Formato dos dados de entrada e saída
│   └── routers/
│       ├── quartos.py
│       ├── hospedes.py
│       ├── reservas.py
│       └── relatorios.py
├── dados.py        # SQLite
└── seed.py         # Quartos de exemplo
tests/
config/settings.json
```
---
## Como executar

```bash
git clone https://github.com/<usuario>/hotel-reservas.git
cd hotel-reservas
python -m venv .venv
source .venv/bin/activate      # No Windows: .venv\Scripts\activate
pip install -e ".[dev]"
python -m hotel.seed           # cria os quartos de exemplo
uvicorn hotel.api.main:app --reload
```

Abra **http://127.0.0.1:8000/docs** para testar todas as rotas direto no navegador (documentação interativa gerada automaticamente pelo FastAPI).

---
## Endpoints

| Método | Rota | O que faz |
| --- | --- | --- |
| POST | `/quartos` | Cadastra um quarto |
| GET | `/quartos` | Lista os quartos |
| GET | `/quartos/{numero}` | Busca um quarto |
| POST | `/quartos/{numero}/bloqueios` | Bloqueia para manutenção |
| POST | `/hospedes` | Cadastra um hóspede |
| GET | `/hospedes/{id}` | Busca um hóspede |
| POST | `/reservas` | Cria uma reserva |
| GET | `/reservas/{id}` | Busca uma reserva |
| POST | `/reservas/{id}/checkin` | Faz o check-in |
| POST | `/reservas/{id}/checkout` | Faz o check-out |
| POST | `/reservas/{id}/cancelamento` | Cancela (com multa, se aplicável) |
| POST | `/reservas/{id}/adicionais` | Adiciona um consumo (ex.: frigobar) |
| POST | `/reservas/{id}/pagamentos` | Registra um pagamento |
| GET | `/relatorios/ocupacao` | Taxa de ocupação no período |
| GET | `/relatorios/revpar` | RevPAR e ADR no período |

---
## Exemplo de uso

Criar uma reserva:

```bash
curl -X POST http://127.0.0.1:8000/reservas \
  -H "Content-Type: application/json" \
  -d '{
    "hospede_id": 1,
    "quarto": 101,
    "entrada": "2026-10-05",
    "saida": "2026-10-08",
    "num_hospedes": 2
  }'
```

Resposta (`201 Created`):

```json
{
  "id": 3,
  "estado": "CONFIRMADA",
  "diarias": 3,
  "valor_total": 780.00
}
```

Consultar a ocupação de outubro:

```bash
curl "http://127.0.0.1:8000/relatorios/ocupacao?inicio=2026-10-01&fim=2026-10-31"
```
---
## Códigos de resposta

| Código | Quando acontece |
| --- | --- |
| 200 | Deu certo |
| 201 | Recurso criado |
| 404 | Quarto, hóspede ou reserva não encontrado |
| 409 | Conflito (quarto já reservado ou bloqueado no período) |
| 422 | Dados inválidos (ex.: saída antes da entrada) |

---
## Configuração

As regras de preço e cancelamento ficam em `config/settings.json`:

```json
{
  "temporadas": [
    {"nome": "Alta", "inicio": "12-20", "fim": "01-05", "multiplicador": 1.3}
  ],
  "fds_multiplicador": 1.1,
  "cancelamento": {"janela_horas": 48, "multa_pct": 0.5}
}
```
---
## Testes

```bash
pytest -v
```

Os testes de serviços e modelos cobrem disponibilidade, capacidade, preços, estados da reserva, cancelamento e relatórios. Os testes da API usam o `TestClient` do FastAPI para chamar as rotas sem subir o servidor.

---