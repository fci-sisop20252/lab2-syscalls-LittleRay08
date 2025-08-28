# 📝 Relatório do Laboratório 2 - Chamadas de Sistema

---

## 1️⃣ Exercício 1a - Observação printf() vs 1b - write()

### 💻 Comandos executados:
```bash
strace -e write ./ex1a_printf
strace -e write ./ex1b_write
```

### 🔍 Análise

**1. Quantas syscalls write() cada programa gerou?**
- ex1a_printf: 9 syscalls
- ex1b_write: 7 syscalls

**2. Por que há diferença entre os dois métodos? Consulte o docs/printf_vs_write.md**

```
A diferença ocorre porque printf() é uma função de biblioteca (stdio) que usa buffers em espaço de usuário, 
ou seja, ele acumula os dados em memória antes de chamar write(). Isso pode resultar em menos chamadas 
ao sistema, mas torna o comportamento dependente do gerenciamento do buffer. Já write() é uma syscall 
direta, que envia os dados imediatamente para o kernel, sem buffering intermediário.

```

**3. Qual método é mais previsível? Por quê você acha isso?**

```
O método mais previsível é o write(), pois cada chamada corresponde exatamente a uma syscall que envia 
os dados para o kernel. O printf(), por usar buffering, pode gerar mais ou menos chamadas dependendo 
do tamanho do buffer, da presença de '\n' ou de flush manual. Assim, o comportamento do write() é 
mais consistente e direto.
```

---

## 2️⃣ Exercício 2 - Leitura de Arquivo

### 📊 Resultados da execução:
- File descriptor: 3
- Bytes lidos: 127

### 🔧 Comando strace:
```bash
strace -e openat,read,close ./ex2_leitura
```

### 🔍 Análise

**1. Qual file descriptor foi usado? Por que não começou em 0, 1 ou 2?**

```
O file descriptor geralmente foi 3, porque 0, 1 e 2 já estão reservados para stdin, stdout e stderr. O próximo arquivo aberto recebe o primeiro número livre, que costuma ser o 3.
```

**2. Como você sabe que o arquivo foi lido completamente?**

```
Sabemos porque o read() retorna 0, que indica EOF (End Of File). Antes disso, os retornos de read trazem a quantidade de bytes realmente lidos.
```

**3. Por que verificar retorno de cada syscall?**

```
Porque qualquer read(), open() ou close() pode falhar. Verificar os retornos evita erros silenciosos (arquivo inexistente, permissões, leitura parcial etc.).
```

---

## 3️⃣ Exercício 3 - Contador com Loop

### 📋 Resultados (BUFFER_SIZE = 64):
- Linhas: 25 (esperado: 25)
- Caracteres: 1300
- Chamadas read(): 21
- Tempo: 0.000100 segundos

### 🧪 Experimentos com buffer:

| Buffer Size | Chamadas read() | Tempo (s) |
|-------------|-----------------|-----------|
| 16          |       82        | 0.001601  |
| 64          |       21        | 0.000438  |
| 256         |       6         | 0.000254  |
| 1024        |       2         | 0.000183  |

### 🔍 Análise

**1. Como o tamanho do buffer afeta o número de syscalls?**

```
Quanto maior o buffer, menos chamadas de read() são necessárias para ler o arquivo inteiro, porque cada chamada traz mais dados de uma vez.
```

**2. Todas as chamadas read() retornaram BUFFER_SIZE bytes? Discorra brevemente sobre**

```
Não. A última leitura quase sempre retorna menos que o tamanho do buffer, pois o arquivo termina antes de encher o buffer.
```

**3. Qual é a relação entre syscalls e performance?**

```
Mais syscalls -> mais tempo gasto no modo kernel.
Buffers maiores -> menos chamadas -> melhor performance.
Mas existe um ponto de equilíbrio, já que buffers gigantes também podem consumir memória à toa.
```

---

## 4️⃣ Exercício 4 - Cópia de Arquivo

### 📈 Resultados:
- Bytes copiados: 1364
- Operações: 6
- Tempo: 0.000393 segundos
- Throughput: 3389.39 KB/s

### ✅ Verificação:
```bash
diff dados/origem.txt dados/destino.txt
```
Resultado: [X] Idênticos [ ] Diferentes

### 🔍 Análise

**1. Por que devemos verificar que bytes_escritos == bytes_lidos?**

```
Para garantir que nada foi perdido na cópia — cada byte lido precisa ser realmente escrito no destino.
```

**2. Que flags são essenciais no open() do destino?**

```
O_WRONLY | O_CREAT | O_TRUNC

- O_WRONLY → abre para escrita

- O_CREAT → cria caso não exista

- O_TRUNC → zera o arquivo antes de escrever (senão sobra lixo)
```

**3. O número de reads e writes é igual? Por quê?**

```
Sim, geralmente é igual, porque o programa copia bloco a bloco: para cada leitura de N bytes, há uma escrita desses mesmos N bytes.
```

**4. Como você saberia se o disco ficou cheio?**

```
O write() retornaria um valor menor do que o esperado (menos bytes escritos do que lidos).
```

**5. O que acontece se esquecer de fechar os arquivos?**

```
- O descritor de arquivo continua aberto no kernel → vazamento de recursos

- Pode não salvar corretamente os dados no disco, já que o buffer do sistema pode não ter sido descarregado (flush).
```

---

## 🎯 Análise Geral

### 📖 Conceitos Fundamentais

**1. Como as syscalls demonstram a transição usuário → kernel?**

```
As syscalls funcionam como uma porta de entrada do processo em espaço de usuário para o kernel. 
Quando o programa chama funções como read(), write(), open(), ele não acessa diretamente o 
hardware, mas solicita ao kernel (via syscall) que execute a operação. Isso demonstra a 
transição porque há uma mudança de contexto: o processador sai do "modo usuário" e entra no 
"modo kernel" para executar instruções privilegiadas.

```

**2. Qual é o seu entendimento sobre a importância dos file descriptors?**

```
Os file descriptors são fundamentais porque funcionam como identificadores numéricos que o kernel 
usa para rastrear arquivos, sockets, pipes e outros recursos de I/O abertos por um processo. Eles 
permitem que o programa interaja com diferentes fontes/destinos de dados de forma unificada, já que 
ler/escrever em um arquivo, na tela ou em um socket segue a mesma interface (read/write).

```

**3. Discorra sobre a relação entre o tamanho do buffer e performance:**

```
Buffers maiores reduzem o número de chamadas ao sistema (syscalls), já que mais dados são lidos ou 
escritos de uma vez só. Isso diminui a sobrecarga de transição usuário→kernel e melhora a performance. 
Por outro lado, buffers muito pequenos aumentam a quantidade de syscalls e tornam o programa mais lento. 
O ideal é encontrar um equilíbrio entre uso de memória e desempenho.
```

### ⚡ Comparação de Performance

```bash
# Teste seu programa vs cp do sistema
time ./ex4_copia
time cp dados/origem.txt dados/destino_cp.txt
```

**Qual foi mais rápido?** O comando cp do sistema.

**Por que você acha que foi mais rápido?**

```
O cp é otimizado em baixo nível, possivelmente utilizando syscalls mais eficientes, técnicas de 
buffering avançadas e até mesmo chamadas específicas do kernel (como sendfile em alguns sistemas). 
Nosso programa didático, por outro lado, usa apenas um loop simples de read/write, sem essas otimizações. 
Isso explica porque o cp atinge um throughput maior.
```

---

## 📤 Entrega
Certifique-se de ter:
- [X] Todos os códigos com TODOs completados
- [X] Traces salvos em `traces/`
- [X] Este relatório preenchido como `RELATORIO.md`

```bash
strace -e write -o traces/ex1a_trace.txt ./ex1a_printf
strace -e write -o traces/ex1b_trace.txt ./ex1b_write
strace -o traces/ex2_trace.txt ./ex2_leitura
strace -c -o traces/ex3_stats.txt ./ex3_contador
strace -o traces/ex4_trace.txt ./ex4_copia
```
# Bom trabalho!
