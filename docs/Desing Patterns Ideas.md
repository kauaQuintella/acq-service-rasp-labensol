# 1. Padrões de Projeto (Design Patterns GoF / Enterprise)
## **Producer-Consumer (Produtor-Consumidor):**

- **Como funciona:** O seu loop de aquisição atua como o **Produtor**, enfileirando os eventos em uma estrutura de dados na memória (como uma `BlockingQueue` ou `Channel`). Uma ou mais threads separadas atuam como **Consumidores**.
- **Momento da escrita:** O consumidor fica lendo essa fila em segundo plano e processando as escritas no banco de dados de forma assíncrona, desatrelando a velocidade de captura da velocidade de escrita.

# 2. Padrões de Persistência e Otimização

- **Batching / Bulk Insert (Agrupamento em Lote):**
    - **Como funciona:** Não é um padrão do GoF, mas uma técnica essencial de persistência. Os eventos são acumulados em uma coleção temporária na memória (um `List` ou `Array`).
    - **Momento da escrita:** Quando a coleção atinge o tamanho ideal para o banco de dados (ex: 500 registros), uma única instrução de inserção em lote (Bulk Insert) é executada.