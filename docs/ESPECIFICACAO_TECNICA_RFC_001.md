# Especificação Técnica de Engenharia: Copiloto de Reservas (RFC-001)

> **Documento de Arquitetura de Software e Engenharia de Backend**  
> **Identificador Técnico:** `QLO-FEAT-001` | **RFC:** `RFC-001`  
> **Componente:** Assistant Service C++17 & Módulo QloApps  
> **Status:** Homologado / Production-Ready  
> **Versão:** 1.0.0  

---

## 🏛️ 1. Visão Geral da Arquitetura

O **Copiloto Observável de Reservas** é implementado sob o padrão **Sidecar Architecture**, desacoplando o processamento de inferência semântica e resolução temporal em um microsserviço C++17 autônomo e determinístico, preservando a integridade do núcleo legado do QloApps (PHP 8.1 / MySQL).

```text
[Atendente / Back-Office QloApps]
                │
                ▼
[Módulo PHP: qloreservationassistant]  <───>  [MySQL: Histórico de Auditoria]
                │
                │ HTTP POST (application/json)
                │ Loopback Local: 127.0.0.1:8101/v1/assist/interpret
                │ Header Obrigatório: X-Correlation-ID: <uuid-v4>
                ▼
[Serviço Local C++17 (Sidecar)]
  ├── Camada HTTP / Roteamento: cpp-httplib (Porta 8101)
  ├── Camada de Validação: RequestValidator (RFC 7807 Problem Details)
  ├── Camada de Domínio: QueryClassifier (Pontuação Heurística Ponderada)
  ├── Camada de Domínio: SlotExtractor (Expressões Regulares e Dicionários)
  └── Motor de Datas: DateResolver (std::tm e mktime determinístico)
```

---

## 📂 2. Estrutura de Diretórios e Módulos

```text
.
├── assistant-service-cpp/               # Microsserviço de Inferência em C++17
│   ├── api/
│   │   └── openapi.yaml                 # Contrato formal OpenAPI 3.1.0
│   ├── include/
│   │   ├── domain/
│   │   │   ├── intent.hpp               # Definição canônica de intenções
│   │   │   ├── query_classifier.hpp     # Classificação semântica e scoring
│   │   │   └── slot_extractor.hpp       # Extração de tipologia, datas e hóspedes
│   │   ├── http/
│   │   │   └── http_server.hpp          # Servidor HTTP cpp-httplib
│   │   ├── validation/
│   │   │   └── request_validator.hpp    # Validação e esquemas RFC 7807
│   │   ├── engine.hpp                   # Orquestrador da inferência
│   │   ├── text_normalizer.hpp          # Normalização UTF-8 / ASCII folding
│   │   ├── httplib.h                    # cpp-httplib (v0.18.3, header-only)
│   │   └── json.hpp                     # nlohmann/json (v3.11.3, header-only)
│   ├── src/
│   │   ├── app.cpp                      # Inicialização da aplicação
│   │   ├── http_server.cpp              # Implementação dos endpoints REST
│   │   └── main.cpp                     # Entrypoint e tratamento de sinais POSIX
│   ├── tests/
│   │   ├── test_classifier.cpp          # 8 testes unitários do classificador
│   │   ├── test_api.cpp                 # 25 testes de integração HTTP / RFC 7807
│   │   ├── run_contract_tests.sh        # Suíte Schemathesis (OpenAPI 3.1)
│   │   ├── k6_load_test.js              # Script k6 de carga (SLA P95 < 20ms)
│   │   └── run_load_test.sh             # Runner automatizado k6
│   ├── Doxyfile                         # Configuração Doxygen para documentação de código
│   └── Makefile                         # Build, compilação, testes e sanitização
│
├── modules/qloreservationassistant/     # Módulo Nativo do QloApps (PHP)
│   ├── qloreservationassistant.php      # Ciclo de vida: install, uninstall, tabs
│   ├── controllers/admin/
│   │   └── AdminReservationAssistantController.php # Controller: cURL, contingência e auditoria
│   ├── views/templates/admin/
│   │   └── assistant_view.tpl           # Template Smarty com cards e tabela de histórico
│   └── tests/
│       ├── test_module.php              # 17 testes unitários do módulo PHP
│       └── e2e/
│           └── test_assistant_e2e.js    # 8 cenários E2E com Playwright (22 asserções)
│
└── docs/
    ├── ESPECIFICACAO_TECNICA_RFC_001.md # Esta especificação técnica
    └── RELATORIO_EXECUTIVO_RFC_001.md   # Relatório executivo de entrega
```

---

## 📡 3. Contrato de API (OpenAPI 3.1 & RFC 7807)

### Endpoint de Inferência
* **Método:** `POST`
* **URL:** `http://127.0.0.1:8101/v1/assist/interpret`
* **Headers Obrigatórios:**
  - `Content-Type: application/json`
  - `X-Correlation-ID: <uuid-v4>`

#### Payload de Requisição (Exemplo):
```json
{
  "query": "Tem quarto deluxe para 2 adultos depois de amanhã?",
  "reference_date": "2026-09-18"
}
```

#### Payload de Resposta de Sucesso (HTTP 200 OK):
```json
{
  "correlation_id": "req-123e4567-e89b",
  "intent": "AVAILABILITY_QUERY",
  "confidence": 0.98,
  "slots": {
    "room_type": "deluxe",
    "check_in": "2026-09-20",
    "check_out": "2026-09-21",
    "guests": 2,
    "policy_category": null,
    "reservation_code": null
  },
  "explanation": "Identificado tipo de quarto 'deluxe', contagem de hospedes (2) e data relativa 'depois de amanha' calculada como 2026-09-20 baseada em 2026-09-18."
}
```

#### Payload de Erro Padronizado (RFC 7807 - HTTP 400 Bad Request):
```json
{
  "type": "https://hotel.local/errors/missing-reference-date",
  "title": "Invalid Request Payload",
  "status": 400,
  "detail": "Field 'reference_date' is required and must match format YYYY-MM-DD.",
  "instance": "/v1/assist/interpret",
  "code": "MISSING_REFERENCE_DATE",
  "invalid_params": [
    {
      "name": "reference_date",
      "reason": "Field 'reference_date' is required and must match format YYYY-MM-DD."
    }
  ]
}
```

---

## 🧠 4. Mecanismo de NLU e Pontuação Ponderada

O motor C++ não utiliza modelos estocásticos em nuvem, garantindo ausência total de alucinações e previsibilidade de custos. A inferência é 100% determinística através de pontuação de evidências léxicas:

1. **Filtro de Desambiguação de Domínio (RN-008):**
   - Consultas citando serviços operacionais (governança: *camareira*, *limpeza*; A&B: *almoço*, *cardápio*; manutenção: *quebrado*) retornam imediatamente `intent = UNKNOWN` com `confidence = 0.35`.
2. **Pontuação Base de Disponibilidade (RN-002):**
   - Presença de verbos/termos de reserva (*"tem vaga"*, *"tem quarto"*, *"reservar"*, *"disponível"*) inicia o score em **`0.85`**.
   - Se o tipo de quarto for omitido, `room_type = null` é retornado sem penalizar a classificação.
3. **Reforço de Entidades:**
   - **Slot Temporal:** Expressões como *"amanhã"*, *"depois de amanhã"* somam **`+0.05`**.
   - **Tipologia Específica:** *deluxe*, *suíte*, *presidencial* somam **`+0.08`**.
   - **Contagem de Hóspedes:** *2 pessoas*, *casal*, *solteiro* somam **`+0.03`**.
   - **Teto:** Pontuação limitada em **`0.98`**.
4. **Aritmética de Datas Relativas:**
   - A partir de `reference_date`, o deslocamento de dias é aplicado via `std::tm` e resolvido com `mktime`, garantindo a correta virada de meses (31/08 $\to$ 01/09), virada de anos (31/12 $\to$ 01/01) e anos bissextos.

---

## 🧪 5. Execução da Suíte de Testes

### Compilação do Serviço C++
```bash
make -C assistant-service-cpp
```

### 1. Testes Unitários e de Integração HTTP (C++)
```bash
make -C assistant-service-cpp test
```
*Executa 8 testes unitários de regressão do classificador + 25 testes de API HTTP e conformidade RFC 7807.*

### 2. Testes de Contrato OpenAPI 3.1 (Schemathesis)
```bash
# Inicie o serviço C++ em segundo plano antes da execução
./assistant-service-cpp/assistant_service &
./assistant-service-cpp/tests/run_contract_tests.sh
```
*Valida 77 cenários de conformidade de schemas e robustez de contrato.*

### 3. Testes de Carga e Validação de SLA (k6)
```bash
./assistant-service-cpp/tests/run_load_test.sh
```
*Simula 20 usuários virtuais concorrentes. Valida SLA de latência (P95 $< 20\text{ ms}$) e taxa de erro ($0\%$).*

### 4. Testes Unitários do Módulo PHP
```bash
php modules/qloreservationassistant/tests/test_module.php
```
*Valida 17 cenários de integração, auditoria e contingência nativa (fallback HTTP 503).*

### 5. Testes End-to-End no Navegador (Playwright)
```bash
# Modo headless (execução em linha de comando)
node modules/qloreservationassistant/tests/e2e/test_assistant_e2e.js

# Modo visual desacelerado para demonstrações
node modules/qloreservationassistant/tests/e2e/test_assistant_e2e.js --headed
```
