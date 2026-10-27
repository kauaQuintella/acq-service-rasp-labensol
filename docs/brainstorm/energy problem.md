Essa é uma excelente preocupação. Como o cartão SD da Raspberry Pi é notoriamente sensível a quedas repentinas de energia (o que pode corromper tanto o banco de dados quanto o próprio sistema operacional), ==precisamos projetar o software para ser **resiliente a falhas**==.

Se a Raspberry Pi desligar do nada, o comportamento dos dados e as estratégias para lidar com isso dependem de onde eles estavam no momento exato da queda.

Abaixo, explico como os dados ficam e quais as melhores técnicas para protegê-los sem sobrecarregar o hardware.

---

1. O que acontece com os dados que estavam no lote (Batch) na memória?

Os dados que estavam na memória RAM dentro da lista aguardando o momento do disparo (o "lote") **serão perdidos**.

Como mitigar essa perda sem detonar o cartão SD?

Para um cenário de medições escalares de poucas amostras por segundo, podemos usar uma estratégia híbrida muito segura:

- **Gatilho Duplo no Batch:** O consumidor grava no banco usando dois critérios: **quantidade** ou **tempo**. Defina limites baixos, por exemplo: grava a cada **20 registros** OU a cada **3 segundos** (o que acontecer primeiro). No pior cenário possível, se a luz cair, o professor perde apenas os últimos 3 segundos de um experimento que pode ter durado minutos ou horas. Para o laboratório, perder 3 segundos de dados escalares geralmente é irrelevante.

---

2. O que acontece com os dados que já foram enviados para o SQLite?

O SQLite é incrivelmente robusto e segue o princípio **ACID** (Atomicidade, Consistência, Isolamento e Durabilidade). Se o comando de escrita já tiver sido finalizado, os dados estão salvos.

O grande perigo de desligar a Pi no meio de uma escrita é a **corrupção do arquivo de banco de dados**.

Como proteger o SQLite contra corrupção?

Existem três configurações simples (comandos `PRAGMA`) que você deve executar assim que abrir a conexão com o SQLite no seu script Python:

1. **Ative o Modo WAL (Write-Ahead Logging):**
    
    sql
    
    ```
    PRAGMA journal_mode = WAL;
    ```
    
    Use o código com cuidado.
    
    Em vez de modificar o arquivo do banco diretamente, o SQLite escreve as alterações em um arquivo separado muito mais leve (o `.db-wal`). Se a energia cair durante a escrita, o banco principal continua intacto. Ao reiniciar, o SQLite olha o arquivo WAL, vê que a escrita foi interrompida e faz um rollback automático. Nada é corrompido.
2. **Configure o Synchronous como NORMAL:**
    
    sql
    
    ```
    PRAGMA synchronous = NORMAL;
    ```
    
    Use o código com cuidado.
    
    No modo WAL, o nível `NORMAL` é o equilíbrio perfeito para a Raspberry Pi. Ele garante que os dados sejam sincronizados no disco nos momentos críticos, protegendo contra corrupção em quedas de energia, mas sem travar a aplicação esperando o cartão SD lento.

---

3. Como fica o estado do experimento ao reiniciar? (A Lógica do Sistema)

Se a Pi desligar no meio de um experimento e depois for ligada novamente, o banco de dados estará íntegro (graças ao WAL), mas o sistema estará em um estado "limbo": o experimento anterior não foi finalizado corretamente (não teve um "clique no botão Parar").

Como resolver a lógica de reinicialização?

No seu banco de dados, você provavelmente terá uma tabela de `experimentos` (id, nome, data_inicio, data_fim, status) e uma tabela de `medicoes` (id, experimento_id, timestamp, valor).

- **Status "Interrompido":** Ao iniciar o script Python no boot da Raspberry Pi, faça uma checagem simples no banco de dados: _"Existe algum experimento com status 'Ativo' ou 'Rodando'?"_
- **A Correção:** Se existir, significa que a Pi caiu durante aquela sessão. O script deve atualizar automaticamente o status daquele experimento para `"Interrompido por falha de energia"` e colocar a `data_fim` como o timestamp da última medição salva daquele experimento.
- Isso garante que, quando o professor abrir a interface web, ele verá o experimento anterior listado corretamente, sabendo exatamente onde ele parou antes da queda.

---

Extra: Protegendo o Hardware (Dica de Ouro para Raspberry Pi)

Como o projeto vai rodar em um laboratório acadêmico onde as pessoas podem puxar a Pi da tomada sem desligar pelo sistema, a maior causa de problemas é o sistema operacional corromper o cartão SD.

Se puder, aplique essa configuração na Raspberry Pi:

- **Overlay File System (Modo Somente Leitura):** O Raspberry Pi OS possui uma ferramenta nativa (no menu `sudo raspi-config` -> _Performance Options_ -> _Overlay File System_). Ela transforma todo o cartão SD em **somente leitura**. O sistema operacional roda na RAM e nada no cartão é modificado.
- **Como o SQLite grava então?** Você pode configurar para que **apenas a pasta onde fica o arquivo do SQLite** tenha permissão de escrita. Assim, o resto do sistema fica 100% blindado contra corrupção induzida por queda de energia.

O que acha dessa estratégia de Gatilho Duplo (tempo/quantidade) aliado ao modo WAL do SQLite? Se quiser, posso te mostrar **como ficaria o código em Python para configurar o SQLite com esses PRAGMAs e fazer o insert em lote seguro**.