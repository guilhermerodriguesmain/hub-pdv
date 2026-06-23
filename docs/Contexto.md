# **Relatório Técnico de Requisitos: API de Gerenciamento de Pedidos Assíncrona**

## **1\. Introdução e Contexto do Projeto**

Este documento estabelece as especificações técnicas para o desenvolvimento da API de Gerenciamento de Pedidos, uma iniciativa central na modernização da infraestrutura de comércio eletrônico da companhia. A transição visa substituir sistemas legados por uma arquitetura orientada a eventos (EDA), priorizando o desacoplamento, a escalabilidade e a consistência eventual.

O objetivo principal é implementar uma solução capaz de realizar o processamento de pedidos de forma assíncrona, utilizando bancos de dados NoSQL para alta disponibilidade e sistemas de mensageria distribuída (RabbitMQ e Kafka) para garantir a integração fluida com sistemas externos e subsistemas de processamento posterior.

## **2\. Stack Tecnológica e Requisitos de Ambiente**

A arquitetura será sustentada pelas seguintes tecnologias, selecionadas por sua performance em ambientes de alta carga:

| Componente | Tecnologia/Ferramenta | Papel na Arquitetura |
| :---- | :---- | :---- |
| Framework de API | **FastAPI** | Interface RESTful assíncrona de alto desempenho. |
| Banco de Dados NoSQL | **MongoDB** | Persistência documental para dados de pedidos. |
| Mensageria (Filas) | **RabbitMQ** | Distribuição de tarefas para consumo interno imediato. |
| Streaming de Eventos | **Kafka** | Barramento de eventos para integração de ecossistema. |
| Coordenação | **Zookeeper** | Gestão de metadados e coordenação do cluster Kafka. |
| Suíte de Testes | **Pytest** | Framework de testes unitários e de integração. |
| Conteinerização | **Docker** | Padronização do ambiente de execução e serviços. |

**Requisitos de Infraestrutura para Desenvolvimento:**

* **Orquestração:** Uso obrigatório do Docker Compose para gerenciamento dos serviços.  
* **Isolamento:** A aplicação deve ser isolada em containers, comunicando-se através de redes virtuais internas do Docker.  
* **Variáveis de Ambiente:** Parâmetros críticos (strings de conexão do MongoDB, URI do RabbitMQ e Bootstrap Servers do Kafka) devem ser injetados via variáveis de ambiente para garantir a portabilidade entre ambientes (Dev/Staging/Prod).

## **3\. Especificações Funcionais e Contrato de Dados**

### **3.1. Cadastro de Pedidos (POST /pedidos)**

O endpoint de cadastro é o ponto de entrada da transação de negócio. A lógica de implementação deve seguir rigorosamente a sequência: **Persistência \-\> Publicação em Fila \-\> Publicação em Tópico.**

* **Identificação:** O backend deve gerar um Identificador Único (UUID v4 ou ObjectId) para cada registro.  
* **Status Inicial:** Todo pedido deve ser criado obrigatoriamente com o status `PENDENTE`.  
* **Schema JSON de Entrada/Saída:**

{  
  "id": "string (uuid)",  
  "nome\_cliente": "string",  
  "nome\_produto": "string",  
  "quantidade": "integer",  
  "status": "PENDENTE"  
}

* **Lógica de Execução:**  
  1. Validar o payload de entrada.  
  2. Persistir o documento na coleção de pedidos do MongoDB.  
  3. Publicar mensagem de sinalização no **RabbitMQ**.  
  4. Publicar evento de criação no **Kafka**.

### **3.2. Consulta de Pedidos (GET /pedidos)**

Este endpoint provê visibilidade sobre o estado atual do sistema.

* **Funcionalidade:** Recuperar e listar todos os documentos armazenados na coleção do MongoDB.  
* **Contrato:** O retorno deve ser uma lista de objetos JSON respeitando o schema definido na seção 3.1.

## **4\. Persistência de Dados e Mensageria**

### **4.1. MongoDB (Persistência Documental)**

Os dados devem ser armazenados em uma coleção denominada `pedidos`. A estrutura do documento deve refletir o contrato da API, garantindo que o esquema NoSQL suporte consultas eficientes por identificador.

### **4.2. RabbitMQ (Comunicação Assíncrona por Fila)**

Utilizado para sinalização interna. A mensagem enviada deve ser um JSON simplificado contendo o `id_pedido` e o `status`, permitindo que workers de processamento (como estoque ou pagamento) iniciem suas tarefas imediatamente.

### **4.3. Kafka (Streaming de Eventos)**

Utilizado para integração com sistemas externos (ex: BI, Logística, Notificações). O evento deve ser publicado em um tópico específico (ex: `order.events`) e deve carregar o payload completo do pedido (PedidoCriado), funcionando como um registro histórico e imutável da transação.

### **4.4. Zookeeper**

Configurado estritamente para suporte e coordenação do cluster Kafka, gerenciando o registro de brokers, eleição de líderes e configurações de tópicos.

## **5\. Plano de Testes e Qualidade**

A validação da API será executada via **Pytest**. Dada a natureza distribuída da aplicação, a estratégia de testes deve contemplar:

* **Mocks de Infraestrutura:** Uso de bibliotecas de "mocking" para simular os clientes do MongoDB, RabbitMQ e Kafka durante os testes unitários, garantindo que a suíte de testes seja rápida e independente de serviços externos.  
* **Fluxo de Cadastro:** Validar se a lógica de geração de IDs e o status `PENDENTE` estão operacionais.  
* **Fluxo de Listagem:** Validar se a integração (ou mock) com o MongoDB retorna a estrutura de dados correta.

## **6\. Infraestrutura e Conteinerização**

A aplicação deve ser orquestrada para garantir que o desenvolvedor possa levantar todo o ecossistema com um comando. O arquivo `docker-compose.yml` deve integrar os serviços: `api`, `mongodb`, `rabbitmq`, `kafka` e `zookeeper`.

**Comando Normativo de Execução:**

docker-compose up \--build

As portas de serviço devem ser mapeadas de forma a evitar conflitos com serviços locais, e os volumes devem ser configurados para o MongoDB para garantir a persistência de dados entre reinicializações dos containers.

## **7\. Roteiro de Implementação e Entregáveis**

A entrega final do projeto deve ser consolidada com os seguintes artefatos:

* \[ \] **Código-fonte da API:** Implementação completa em Python/FastAPI seguindo boas práticas de codificação.  
* \[ \] **Configurações de Serviços:** Arquivos de inicialização e ambiente (`.env.example`) contendo as definições para MongoDB, RabbitMQ e Kafka.  
* \[ \] **Suíte de Testes:** Scripts Pytest com cobertura dos fluxos de cadastro e listagem.  
* \[ \] **Manifestos Docker:** `Dockerfile` otimizado para a aplicação e `docker-compose.yml` para orquestração total.  
* \[ \] **Documentação de Contrato:** Definição clara dos endpoints (FastAPI Swagger/OpenAPI).

