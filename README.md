# Projeto-ERAD-RS

## 📋 Pré-requisitos
Antes de iniciar, certifique-se de ter instalado em sua máquina ou servidor:
* **Python 3.10+**
* **Git**
* Acesso a uma GPU com pelo menos 24GB de VRAM

---

<br>

## 🛠️ Tecnologias e Ferramentas
* **Motor de Inferência:** [vLLM](https://github.com/vllm-project/vllm)
* **Modelo:** Meta-Llama-3-8B-Instruct (via *NousResearch*)
* **Banco Vetorial:** FAISS
* **Embeddings:** `sentence-transformers/all-MiniLM-L6-v2`
* **Gerador de Ataque:** [Garak](https://github.com/leondz/garak) (probe `promptinject`)

---

<br>

## 🚀 Passo a Passo

Antes de seguir os comandos, certifique de isolar a infraestrutura em **dois terminais** simultaneamente.

<br>

### 1. Preparando o Ambiente 

Crie um ambiente virtual:

```bash
python -m venv venv
source venv/bin/activate
```

<br>

Instale também o **vLLM**, o **Garak** e as bibiliotecas que serão utilizadas:

```bash
pip install vllm garak faiss-cpu sentence-transformers requests
```

---

<br>

### 2. Servidor LLM *(Terminal 1)*
> Este será o terminal que irá rodar o mortor da IA, hospedeiro do ataque.

```bash
python -m vllm.entrypoints.openai.api_server \
  --model NousResearch/Meta-Llama-3-8B-Instruct \
  --gpu-memory-utilization 0.85 \
  --max-model-len 8192 \
  --enforce-eager
```

<br>

Quando o terminal exibir a mensagem "*Application startup complete*", o vLLM estará rodando perfeitamente, e você poderá continuar com os passos seguintes.


<br>

Analisando o que cada comando faz:

* **vllm.entrpoint.openai.api_server:** Inicia o motor de inferênncia do vLLM, imitando a API da OpenAI. Para o Garak, é como se ele estivesse atacando o GPT-4.
* **--model NousResearch/Meta-Llama-3-8B-Instruct:** Indica o repositório do ***HuggingFace** ao vLLM para baixar e carregar a memória da GPU. O NousResearch funciona como uma "versão liberada" do modelo LLama-3 sem que passe por um processo de autorização da Meta.
* **--gpu-memory-utilization 0.85:** Reserva exatamente 85% de toda a VRRAM disponível na GPU logo na inicialização.
* **--max-model-len 8192:** Limita a quantidade de tokens que o modelo vai processar
* **--enforce-eager:** Desativa a otimização de "CUDA Graphs" do vLLM e força o PyTorch a executar as operações de rede neural no modo *Eager execution*, poupando VRAM.

---

<br>

### 3. Gerando os *payloads* maliciosos com o Garak *(Terminal 2)*

Antes de rodar o teste de desempenho, é preciso gerar o arquivo com as memórias infectadas. Com o servidor rodando no *Terminal 1*, abra o **Terminal 2**, ative a venv e execute o Garak:


```bash
python -m garak --model_type openai --model_name NousResearch/Meta-Llama-3-8B-Instruct --probes promptinject --report_prefix ataques_memoria
```

<br>

O Garak farpa uma varredura enviando testes de segurança para a LLM local. Ao final, gerará um arquivo de relatório chamado "**ataques_memoria.report.jsonl**". É deste arquivo que nosso script Python extrairá os ataques.

> Certifique-se de que o arquivo **.jsonl** está na mesma pasta que o teste está sendo realizado, o garak pode acabar criando uma pasta própria para ele.

---

<br>

### 4. Experimento de Desempenho *(Terminal 2)*

Com o arquivo **.jsonl** gerado e o servidor do *(Terminal 1)* ainda ativo, faça:

```bash
python teste.py
```

<br>

**Como o codigo em Python funciona?**

* **Fase 1 (Baseline)**: Enche o banco FAISS com memórias "limpas" e medee o desempenho de uma interação normal.
* **Fase 2 (Ataque)**: Lê os *payloads* maliciosos e injeta no FAISS, realizando por fim, uma medição de desempenho após o envenenamento de contexto.
