# SISTEMA DE RESERVAS DE HOTEL POR WELLINGTON D. ANGELO
Projeto individial da disciplina ES0008 -Programação Orientada a Objetos ministrada no semestre 2026.2

 ---
## Objetivo
O projeto consiste em desenvolver uma API de um sistema de reservas de hotel. Este permitirá acesso aos dados de hóspedes, quartos e reservas com check-in/check-out. Também devendo conter tratamentos para política de cancelamento, tarifas por temporada, bloqueios por manutenção e relatórios de desempenho.

---

## Estrutura planejada de classes
```
app/
├── main.py
│
├── models/
│   ├── pessoas.py
│   │   ├── Pessoa
│   │   └── Hospede
│   │
│   ├── quartos.py
│   │   ├── Quarto
│   │   ├── QuartoSimples
│   │   ├── QuartoDuplo
│   │   └── QuartoLuxo
│   │
│   ├── registros.py
│   │   ├── Reserva
│   │   ├── Pagamento
│   │   ├── Adicional
│   │   └── Bloqueio
│   │
│   └── auditavel.py
│       └── Rastreabilidade
│
├── services/
│   └── relatorios.py
│
└── routes/
    ├── hospedes.py
    ├── quartos.py
    └── reservas.py
```
---
## UML TEXTUAL

```
                         ┌──────────────────────┐
                         │       Pessoa         │
                         ├──────────────────────┤
                         │ - nome: str          │
                         │ - documento: str     │
                         │ - email: str         │
                         │ - telefone: str      │
                         ├──────────────────────┤
                         │- criar_reserva()     │
                         │- cancel_reserva()    │
                         └──────────△───────────┘
                                    │
                         ┌──────────┴──────────┐
                         │                     │
                      herança               herança
                         │                     │
              ┌──────────┴───────┐   ┌─────────┴─────────┐
              │     Hospede      │   │    Funcionario    │
              ├──────────────────┤   ├───────────────────┤
              │ - nome: str      │   │ - nome: str       │
              │ - documento: str │   │ - documento: str  │
              │ - email: str     │   │ - email: str      │
              │ - telefone: str  │   │ - telefone: str   │
              ├──────────────────┤   │ - matricula: int  │
              │ - reservas       │   │ - cargo: str      │
              │   : list[Reserva]│   ├───────────────────┤
              │- criar_reserva() │   │- criar_reserva()  │
              │- cancel_reserva()│   │- cancel_reserva() │
              └────────┬─────────┘   │- block_apto()     │
                       │             │- reg_check-in()   │
                       │             │- reg_check-out()  │
                       │             │- reg_pagamento()  │
                       │             │- reg_adicional()  │
                       │             └─────────┬─────────┘
                       │                       │
                       │ 1                     │
                       │                       │
                       │ 0..*                  │
                       ▼                       │
              ┌──────────────────┐             │
              │     Reserva      │◄────────────┤
              ├──────────────────┤   gerencia/ │
              │ - id             │   realiza operações
              │ - data_entrada   │             │
              │ - data_saida     │             │
              │ - qtd_hospedes   │             │
              │ - origem         │             │
              │ - status         │             │
              │ - pagamentos     │             │
              │ - adicionais     │             │
              └────────┬─────────┘             │
                       │                       │
             ┌─────────┴──────────┐            │
             │                    │            │
       composição            composição        │
             │                    │            │
             ▼                    ▼            │
     ┌──────────────┐     ┌──────────────┐     │
     │  Pagamento   │     │  Adicional   │     │
     ├──────────────┤     ├──────────────┤     │
     │ - data       │     │ - descricao  │     │
     │ - forma      │     │ - valor      │     │
     │ - valor      │     └──────────────┘     │
     └──────────────┘                          │
                                               │
                                               │
              ┌──────────────────┐             │
              │      Quarto      │             │
              ├──────────────────┤             │
              │ - numero         │             │
              │ - capacidade     │             │
              │ - Diária         │             │
              │ - status         │             │
              │ - bloqueios      │ ◄───────────┤
              └────────△─────────┘ Bloquear/Alterar Status
                       │                       
             ┌─────────┼──────────┐
             │         │          │
          herança   herança    herança
             │         │          │
             ▼         ▼          ▼
       ┌──────────┐ ┌──────────┐ ┌──────────┐
       │  Simples │ │  Duplo   │ │   Luxo   │
       └──────────┘ └──────────┘ └──────────┘
```
