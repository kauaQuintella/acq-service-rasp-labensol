
O escopo real do seu projeto mudou completamente o cenário (e para melhor!). Como o foco são **medições escalares** (como Vpp, Mean) em uma taxa de **poucas medidas por segundo**, e não formas de onda brutas a 30 fps, a restrição de hardware da Raspberry Pi 3 (906 MiB RAM, escrita em Cartão SD) deixa de ser um problema crítico e passa a ser algo facilmente contornável com padrões simples.

Para o seu cenário, o padrão ideal é o **Producer-Consumer (Produtor-Consumidor)** com um mecanismo de **Batching (Escrita em Lote)** simples na memória.

Aqui está o desenho ideal para a arquitetura do seu projeto, considerando as restrições da Raspberry Pi 3:

---

A Arquitetura Recomendada: Produtor-Consumidor Assíncrono

Você deve separar o seu programa em três partes (threads/tarefas) independentes que rodam na Pi:

```
[ Thread 1: Loop de Aquisição ] 
             │ (Produz JSON/Dict)
             ▼
      [ Fila em Memória ] ───► [ Thread 3: WebSocket Server (FastAPI) ] ──► (Gráfico ao vivo no Navegador)
             │
             ▼
[ Thread 2: Consumidor do Banco ] ──► [ Acumulador/Batcher ] ──► [ SQLite (Cartão SD) ]
```

1. O Produtor (Loop de Aquisição)

Uma thread ou tarefa `asyncio` dedicada exclusivamente a conversar com o osciloscópio via `PyVISA`.

- Ela roda no intervalo desejado (ex: a cada 200ms ou 500ms).
- Ela lê os valores escalares, monta um dicionário Python (JSON) ex: `{"timestamp": 171822..., "ch1_vpp": 3.42, "ch2_mean": 1.12}`.
- Ela joga esse dicionário em **duas filas em memória** (uma para o banco de dados e outra para a interface web).

2. O Consumidor de Banco + Batching (Por que isso é vital na RPi 3?)

Cartões SD possuem uma vida útil limitada e uma velocidade de escrita aleatória **muito lenta**. Fazer um `INSERT` no SQLite a cada fração de segundo vai travar a sua aplicação devido ao I/O do cartão SD e pode corrompê-lo rapidamente.

- **A Solução:** O seu consumidor lê a fila do banco e acumula esses JSONs em uma lista na memória.
- **O Gatilho:** Quando a lista atingir **50 registros** (ou a cada **10 segundos**, o que acontecer primeiro), ele abre uma única transação no SQLite e faz um `executemany()` (Bulk Insert).
- Isso reduz drasticamente o estresse no cartão SD e consome quase nada de RAM.

3. O Consumidor da Interface (Visualização ao Vivo)

Para o gráfico ao vivo no navegador do professor, você não deve ler do SQLite.

- A thread do servidor Web (usando **FastAPI** ou **Flask-SocketIO**) consome a segunda fila em memória e dispara o dado imediatamente via **WebSocket** para o front-end (HTML/JavaScript). O gráfico atualiza instantaneamente sem encostar no cartão SD.

---

Sugestão de Stack Tecnológica (Leve e Modular)

Para respeitar os ~768 MiB livres de RAM e manter o código modular para o futuro:

- **Back-end: FastAPI.** É extremamente rápido, consome pouquíssima RAM (~30-50 MB), possui suporte nativo a `asyncio` (perfeito para loops de medição) e já vem com suporte a WebSockets para o gráfico ao vivo.
- **Banco de Dados: SQLite.** Como os dados são escalares, o banco vai crescer muito devagar. Ative o modo **WAL (Write-Ahead Logging)** no SQLite (`PRAGMA journal_mode=WAL;`), que permite ler os experimentos passados enquanto o consumidor está escrevendo o experimento atual sem travar o banco.
- **Front-end: HTML5 + JavaScript Puro (Vanilla JS) + Chart.js (ou Plotly).** Não use frameworks pesados como React ou Angular se não tiver experiência, pois eles adicionam complexidade desnecessária. Uma página HTML simples servida pelo FastAPI, usando **Chart.js** para renderizar o gráfico de linha ao vivo, roda direto no navegador do professor (o processamento do gráfico fica no PC dele, aliviando a Raspberry Pi).

Por que essa abordagem resolve suas 3 dores?

1. **Autonomia do professor:** O FastAPI serve a página Web. O professor só precisa digitar o IP da Pi no navegador (ex: `http://192.168.1.50`). Vai ter um botão "Iniciar Experimento" e "Parar".
2. **Modularidade:** Se amanhã você mudar o modelo do osciloscópio, você só altera a classe do "Produtor" (Driver PyVISA). O resto do sistema (fila, banco, web) nem sabe que o instrumento mudou. Se quiser adicionar mais canais, basta o dicionário JSON do produtor conter mais chaves.
3. **Organização:** Os dados ficam protegidos no SQLite. Quando o professor quiser um CSV, a rota do FastAPI pode gerar o arquivo `.csv` sob demanda a partir do banco e iniciar o download no navegador dele na hora.

Para avançarmos no desenho do código, você prefere que eu monte um **exemplo prático da estrutura de pastas modular** para o projeto ou um **esboço em Python de como implementar a fila Produtor-Consumidor com Batching para o SQLite**?