# Suíte de Testes de Desempenho – Ecommerce Checkout API

Este projeto destina-se à construção e execução de uma suíte de testes de performance sobre uma API de Checkout de E-commerce (simulada). O foco não é evoluir ou otimizar o código da API, mas atuar como Engenheiro(a) de Performance para caracterizar o comportamento, limites e ponto de ruptura (breaking point) do sistema sob teste (SUT).

## 1. Objetivos de Aprendizagem

- Configurar e executar diferentes tipos de testes de performance com k6.
- Diferenciar na prática testes de Carga (Load), Estresse (Stress), Pico (Spike) e (menção conceitual) Resistência (Soak) mesmo que não implementado aqui.
- Analisar métricas de latência (p95, p99) em relação ao throughput (requisições/segundo).
- Identificar o ponto de ruptura da aplicação (crescimento abrupto de latência / aumento de erros / timeouts).
- Formular critérios (SLA / SLO / Thresholds) e verificar conformidade.

## 2. Sistema Sob Teste (SUT)

O projeto `ecommerce-checkout-api` expõe endpoints com características distintas propositalmente:

- `GET /health`: Resposta rápida (baseline de disponibilidade / verificação simples).
- `POST /checkout/simple`: Simula operação leve de I/O (ex.: interação com banco de dados).
- `POST /checkout/crypto`: Simula operação pesada de CPU (cálculo de hash / criptografia).

Essas diferenças permitem observar comportamentos sob cargas comparáveis e contrastar gargalos de CPU vs. I/O.

## 3. Estrutura do Repositório

```
README.md
atividade/
	package.json
	README.md (secundário, se aplicável)
	src/
		server.js
		tests/
			load.js
			smoke.js
			spike.js
			stress.js
exemplo/
	load-test.js
	package.json
	server.js
	stress-test-k6.js
	user-flow.yml
```

Os scripts definitivos devem residir em `atividade/src/tests/` conforme solicitado.

## 4. Pré-Requisitos

1. Node.js (versão compatível com o projeto; validar via `node -v`).
2. k6 instalado localmente.
   - Windows (Chocolatey): `choco install k6`
   - Windows (Winget): `winget install k6`
   - macOS (Homebrew): `brew install k6`
   - Linux (Debian/Ubuntu): `sudo apt install k6` (ou repositório oficial k6)
3. Acesso ao repositório base: `git clone https://github.com/CleitonSilvaT/teste-de-desempenho.git` (ou este fork).
4. Porta default (verificar `server.js`, usualmente `3000` ou definida por variável de ambiente).

## 5. Configuração da API (SUT)

Dentro da pasta principal da API:

```
npm install
npm start
```

Validar saúde inicial:

```
curl http://localhost:3000/health
```

Resposta esperada: status 200 e corpo simples (ex.: `{ status: "ok" }`).

## 6. Metodologia de Teste

Cada tipo de teste tem objetivo distinto:

- Smoke: Verificar disponibilidade básica antes de execuções extensas.
- Load: Validar comportamento em carga esperada (cenário realista de negócio).
- Stress: Empurrar além do normal para observar degradação e ponto de ruptura.
- Spike: Avaliar reação a surtos abruptos de usuários (flash sale).

Métricas principais:

- Latência: médias, p95, p99.
- Throughput: requisições/segundo (RPS) sustentado e máximo.
- Taxa de erros: percentual de falhas (timeouts, HTTP 5xx, etc.).
- Utilização de recursos (CPU / memória) – observada externamente (top / Task Manager / ferramentas de monitoramento).

Critérios de observação para ruptura:

- Crescimento súbito e persistente do p95/p99.
- Aumento progressivo de erros (HTTP 500 / falhas de conexão).
- Timeouts ou interrupções do processo.

## 7. Etapas e Scripts

### 7.1 Smoke Test (`smoke.js`)

Objetivo: Validar se o serviço está operacional antes dos testes pesados.
Configuração:

- 1 VU ativo por 30s acessando `GET /health`.
  Critério de Sucesso:
- 100% de requisições com status 2xx.

Exemplo de estrutura (conceitual):

```js
import http from "k6/http";
import { check } from "k6";

export const options = {
  vus: 1,
  duration: "30s",
  thresholds: { http_req_failed: ["rate==0"] },
};

export default function () {
  const res = http.get("http://localhost:3000/health");
  check(res, { "status 200": (r) => r.status === 200 });
}
```

### 7.2 Teste de Carga (`load.js`)

Contexto: Marketing prevê pico de 50 usuários simultâneos em promoção.
Endpoint: `POST /checkout/simple`.
Stages:

1. Ramp-up: 0 → 50 VUs em 1m.
2. Platô: 50 VUs por 2m.
3. Ramp-down: 50 → 0 VUs em 30s.
   Thresholds (SLA):

- `http_req_duration{p(95)} < 500ms`
- Erros (falhas) < 1%.

Exemplo (trecho de options):

```js
export const options = {
  stages: [
    { duration: "1m", target: 50 },
    { duration: "2m", target: 50 },
    { duration: "30s", target: 0 },
  ],
  thresholds: {
    http_req_failed: ["rate<0.01"],
    http_req_duration: ["p(95)<500"],
  },
};
```

### 7.3 Teste de Estresse (`stress.js`)

Pergunta: Quantos usuários em operações CPU-heavy (hash) degradam o sistema?
Endpoint: `POST /checkout/crypto`.
Carga agressiva:

1. 0 → 200 VUs em 2m.
2. 200 → 500 VUs em 2m.
3. 500 → 1000 VUs em 2m.
   Observação: Monitorar momento em que latência p95/p99 dispara ou ocorrem timeouts. Registrar manualmente o ponto percebido.

### 7.4 Teste de Pico (`spike.js`)

Objetivo: Simular flash sale (variação súbita).
Endpoint: `POST /checkout/simple`.
Stages:

1. 10 VUs por 30s.
2. Salto imediato para 300 VUs em 10s.
3. Manter 300 VUs por 1m.
4. Queda imediata para 10 VUs.
   Avaliação: Capacidade de absorver pico sem falhas prolongadas e recuperar latência rapidamente.

## 8. Execução dos Testes

Assumindo scripts em `atividade/src/tests/`:

```
cd atividade/src/tests/
k6 run smoke.js
k6 run load.js
k6 run stress.js
k6 run spike.js
```

Recomenda-se executar inicialmente o smoke, depois load, então stress e finalmente spike para comparação.

## 9. Coleta e Interpretação de Métricas

Saída padrão do k6 inclui:

- `http_req_duration`: Distribuição e percentis.
- `http_req_failed`: Taxa de falhas.
- `iterations` / `vus` / `vus_max`.

Interpretação:

- p95 vs p99: p99 mais sensível a caudas; crescimento do p99 antes do p95 sugere início de saturação.
- Throughput sustentado: comparar fases de platô com ramp-up para detectar limitações.
- Erros: correlação com saturação de CPU (estresse) ou fila de I/O (carga).

## 10. Identificação do Breaking Point

Critérios típicos:

- Latência p95 > SLA por 3 intervalos consecutivos.
- Erros > SLA (ex.: >1% load / >5% stress).
- Aumento contínuo de tempo médio sem estabilização ao longo do platô.
  Documentar: VU aproximado, métricas no momento, comportamento observado (ex.: quedas de conexão).

## 11. Relato de Resultados (Resumo Consolidado k6)

### 11.1 Visão Comparativa

| Teste  | Script      | VUs Máx | Duração (aprox.) | p95     | p99     | RPS (aprox.) | Erros (%) | Status predominante | Conclusão curta  |
| ------ | ----------- | ------- | ---------------- | ------- | ------- | ------------ | --------- | ------------------- | ---------------- |
| Smoke  | `smoke.js`  | 1       | 1m               | ~1 ms   | ~1 ms   | ~4275        | 0         | 200 OK              | Saúde confirmada |
| Load   | `load.js`   | 50      | ~4m              | ~294 ms | ~313 ms | ~32          | 0         | 201 Created         | Estável em carga |
| Spike  | `spike.js`  | 300     | ~2m              | ~293 ms | ~313 ms | ~128         | 0         | 201 Created         | Pico absorvido   |
| Stress | `stress.js` | 1000    | ~6m (estágios)   | ~4.43 s | ~49 s   | ~359         | 95.27     | 201 (4% sucesso)    | Ruptura ~200–300 |

### 11.2 Smoke Test (`smoke.js`)

- Objetivo: Verificar disponibilidade básica.
- Configuração: 1 VU por 1 minuto.
- Thresholds: `checks: rate==1`, `http_req_failed: rate==0`.
- Resultados: p95 ~1 ms; RPS ~4275; 0% erros; 100% 200 OK.
- Conclusão: Resposta mínima extremamente rápida; base confiável para prosseguir.

### 11.3 Load Test (`load.js`)

- Objetivo: Validar comportamento com 50 usuários simultâneos.
- Configuração: Ramp-up 1m, platô 2m, ramp-down 30s.
- Thresholds: `p(95)<500ms`, `http_req_failed<1%` (ambos atendidos).
- Resultados: p95 ~294 ms; p99 ~313 ms; RPS ~32; 0% erros; 100% 201.
- Conclusão: Estável; latências dentro do SLA; sem falhas observadas.

### 11.4 Spike Test (`spike.js`)

- Objetivo: Avaliar reação a salto abrupto (flash sale).
- Configuração: 10 VUs (30s) → 300 VUs (10s) → manter 1m → queda para 10 VUs.
- Threshold: `http_req_failed<5%` (0%).
- Resultados: p95 ~293 ms; p99 ~313 ms; RPS ~128; 0% erros; 100% 201.
- Conclusão: Pico absorvido sem degradação; latência permanece similar ao teste de carga.

### 11.5 Stress Test (`stress.js`)

- Objetivo: Determinar ponto de ruptura em workload CPU-heavy.
- Configuração: Escalonamento progressivo até 1000 VUs (0→200, 200→500, 500→1000, cada ~2m).
- Threshold definido: `http_req_failed < 20%` (violado: 95%).
- Resultados: p95 ~4.43 s; p99 ~49 s; RPS ~359; 95.27% erros; apenas ~4% respostas 201 bem-sucedidas.
- Ponto de ruptura: entre ~200 e 300 VUs inicia saturação (crescimento exponencial de latência e falhas).
- Sintomas: fila crescente, explosão do p99, timeouts/refusals, CPU-bound evidente, queda drástica de sucesso.
- Conclusão: Capacidade sustentável antes de degradação severa <300 VUs para operações intensivas de CPU.

### 11.6 Síntese Geral

- Workload leve (`/checkout/simple`): suporta picos transitórios (300 VUs) sem degradação significativa.
- Workload pesada (`/checkout/crypto`): saturação acentuada a partir de ~200–300 VUs.
- Indicadores de ruptura: aumento abrupto p99, taxa de erros >90%, queda em respostas 201.
- Recomendações: otimização de função de criptografia, paralelização ou offloading; considerar escalabilidade horizontal; definir limites operacionais formais.

## 12. Observações

Para garantir resultados confiáveis e reprodutíveis, recomenda-se isolar o ambiente de execução, evitando a execução de outros processos pesados durante os testes, bem como repetir os cenários para validar a consistência dos resultados obtidos. Além disso, é importante versionar os scripts de teste, assegurando controle de mudanças e reprodutibilidade, e manter parâmetros como endpoints e payloads separados em constantes, de forma a facilitar ajustes, manutenção e evolução dos testes.

## 14. Referências

- Documentação k6: https://k6.io/docs/
- Conceitos de Performance Engineering (latência, throughput, percentis, SLA/SLO).
