# DocuViagem ✈️ 🔒 — Sistema Seguro de Coleta de Documentos (Showcase)

> **Aviso:** Este é um repositório de demonstração (*showcase*). O código-fonte original é mantido em um repositório privado por questões de segurança e confidencialidade da agência de viagens cliente.

## 🎯 O Problema

Você mandaria a foto do seu cartão de crédito pelo WhatsApp? O cliente do seu negócio também tem medo de fazer isso.

Durante o planejamento de um projeto para o meu portfólio, um empreendedor (dono de uma agência de viagens) me relatou um problema real: ele estava perdendo vendas porque os clientes se recusavam a enviar documentos sensíveis (RG, CPF, Passaporte) e dados de pagamento por aplicativos de mensagens comuns. Parei os projetos paralelos para resolver essa dor.

## 💡 A Solução

O **DocuViagem** é uma plataforma web *white label* que permite às agências de viagens coletarem documentos de clientes e dados de cartões de crédito com segurança de nível bancário, criptografia ponta a ponta e total conformidade com a LGPD.

[▶️ **CLIQUE AQUI PARA VER O VÍDEO DO SISTEMA FUNCIONANDO**](https://www.linkedin.com/feed/update/urn:li:activity:7481316393901207552/)

---

## 🛠️ Stack Tecnológica

- **Backend:** Node.js 18+ com Express 4
- **Banco de Dados:** MySQL 5.7 (Pool de conexões)
- **Segurança:** Criptografia AES-256-CBC (`crypto` nativo), JWT (Access/Refresh Tokens), Helmet e Rate Limiting
- **Frontend:** SPA construída com HTML5, CSS3 e JavaScript puro (comunicação via Fetch API)
- **Infra/Outros:** Upload de arquivos com Multer, envio de e-mails com Gmail API via Conta de Serviço

---

## 🚀 Arquitetura e Destaques Técnicos

A melhor parte deste projeto foi fechar brechas de segurança e ver o código impactando a operação de um negócio real.

### 1. Criptografia de Ponta a Ponta e Arquitetura de Chaves
Como lidamos com dados críticos, a segurança não poderia depender de uma única chave mestre do servidor. Quando o cliente finaliza o envio dos formulários, o sistema gera dinamicamente uma senha única de 6 caracteres que é exibida **apenas para ele** na tela. 
- Os dados são encriptados utilizando o algoritmo **AES-256-CBC**, usando essa senha do cliente como base da chave.
- **Nem mesmo o administrador** da agência consegue acessar os documentos ou o número do cartão sem que o cliente informe essa senha gerada. O banco de dados armazena apenas os metadados criptografados e o IV (Initialization Vector).

### 2. Validação Real e Prevenção de Fraudes
Para evitar envios sujos, o formulário do cartão não depende de máscaras superficiais. Implementei a lógica de validação do algoritmo oficial matemático da Receita Federal para validar se o CPF é real antes de aceitar a submissão.

### 3. Conformidade com a LGPD (Lei Geral de Proteção de Dados)
- **Termo de Consentimento:** O sistema registra o aceite explícito do usuário, salvando IP, User-Agent e Timestamp antes de qualquer submissão.
- **Exclusão Física Automatizada:** Após o administrador desbloquear e visualizar os documentos, o sistema exclui automaticamente os arquivos (`.jpg`, `.pdf`) do *storage*.
- **Log de Auditoria:** Foi implementada uma tabela de log completa para registrar tentativas de login, acessos a dados sensíveis, alterações e downloads por parte dos administradores.

### 4. Arquitetura White Label
O sistema foi projetado para ser facilmente replicável para outras agências. Toda a parte visual (cores primárias, logos, regras e textos do termo de uso) é gerenciada via um arquivo `config.json` processado pelo `theme-loader.js` no client-side, permitindo personalização completa sem necessidade de refatorar HTML/CSS.

---

## 👨‍💻 Autor

**Antonio Kerr**
- [LinkedIn](https://www.linkedin.com/in/antoniokerr/)
- [GitHub](https://github.com/tonykerr07)
