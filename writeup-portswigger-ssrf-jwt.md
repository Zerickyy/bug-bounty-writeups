# Writeup: PortSwigger Web Security Academy - SSRF e JWT Attacks

## Ambiente
- Atacante: Kali Linux (VM VirtualBox)
- Alvo: Laboratórios do PortSwigger Web Security Academy (hospedados remotamente)
- Ferramentas: Burp Suite (Proxy, Intruder, Repeater, extensão JWT Editor)

---

## Parte 1 — SSRF (Server-Side Request Forgery)

### Lab 1: Basic SSRF against the local server (Apprentice)
Funcionalidade de "check stock" aceita URL completa via parâmetro
`stockApi`, sem validação. Exploração:

    stockApi=http://localhost/admin
    stockApi=http://localhost/admin/delete?username=carlos

Resultado: acesso ao painel administrativo e exclusão de usuário
via requisição forjada pelo próprio servidor (bypass de controle
de acesso baseado em origem "confiável" por vir do loopback).

### Lab 2: Basic SSRF against another back-end system (Apprentice)
Mesma vulnerabilidade, mas o alvo é um sistema interno na rede
privada (192.168.0.0/24), não o próprio servidor. Uso do Burp
Intruder para escanear a faixa de IP na porta 8080:

    stockApi=http://192.168.0.§1§:8080/admin

IP 192.168.0.1 identificado como único com resposta HTTP 200
(demais retornaram erro de conexão). Requisição enviada ao Burp
Repeater para deletar o usuário via:

    stockApi=http://192.168.0.1:8080/admin/delete?username=carlos

### Lab 3: SSRF with blacklist-based input filter (Practitioner)
Aplicação bloqueia duas camadas separadamente: o IP (`127.0.0.1`,
`localhost`) e o caminho (`/admin`). Bypasses:

    http://127.1/                          -> bypass do filtro de IP
                                               (notação abreviada)
    http://127.1/%2561dmin                 -> bypass do filtro de path
                                               (duplo URL-encoding da
                                               letra "a": %2561 decodifica
                                               uma vez para %61, e uma
                                               segunda vez no servidor
                                               real para "a")

Payload final:

    stockApi=http://127.1/%2561dmin/delete?username=carlos

### Lab 4: SSRF with whitelist-based input filter (Expert)
Aplicação valida que o hostname da URL corresponde exatamente a
`stock.weliketoshop.net`. Bypass usando sintaxe de credenciais
embutidas na URL (`usuario@host`) combinada com fragmento (`#`)
duplamente codificado:

    http://username@stock.weliketoshop.net/       -> aceito (parser
                                                      reconhece o host
                                                      esperado)
    http://username%2523@stock.weliketoshop.net/  -> "Internal Server
                                                      Error" (indica
                                                      tentativa de
                                                      conexão a
                                                      "username")

Payload final:

    stockApi=http://localhost:80%2523@stock.weliketoshop.net/admin/delete?username=carlos

O parser da whitelist identifica `stock.weliketoshop.net` como o
host (aprovando a URL), mas o servidor real, ao decodificar
%2523 -> %23 -> "#", interpreta tudo após o "#" como fragmento,
conectando de fato apenas a `localhost:80`.

---

## Parte 2 — JWT Attacks

### Lab 1: JWT authentication bypass via unverified signature (Apprentice)
Servidor decodifica o JWT sem verificar a assinatura. Edição
direta do payload via extensão JWT Editor no Burp Repeater:

    "sub": "wiener"  ->  "sub": "administrator"

Requisição enviada sem recalcular a assinatura — aceita
normalmente, concedendo acesso ao painel /admin.

### Lab 2: JWT authentication bypass via flawed signature verification (Apprentice)
Servidor aceita tokens não assinados (alg: none). Uso da função
"Attack -> None" da extensão JWT Editor, após editar o payload
para "sub": "administrator". Erro inicial identificado: JSON
malformado (aspa de fechamento ausente no valor do campo sub),
corrigido antes do reenvio bem-sucedido.

### Lab 3: JWT authentication bypass via weak signing key (Practitioner)
Servidor usa uma chave secreta HS256 fraca. Chave quebrada por
força bruta offline:

    john --format=HMAC-SHA256 --wordlist=jwt.secrets.list jwt_token.txt

(hashcat indisponível por falta de suporte a GPU na VM,
substituído por John the Ripper, mesma solução usada no crack de
hash MD5 do Dia 1)

Resultado: chave identificada como "secret1" em segundos.
Chave cadastrada na aba "JWT Editor Keys" do Burp como chave
simétrica, usada para re-assinar um token modificado
("sub": "administrator") com assinatura HMAC-SHA256 válida,
concedendo acesso legítimo (do ponto de vista do servidor) ao
painel administrativo.

---

### Lab 4: JWT authentication bypass via jwk header injection (Practitioner)
Servidor confia em chaves públicas embutidas diretamente no
header do JWT (campo "jwk"), sem verificar se vieram de uma
fonte confiável. Gerada uma chave RSA própria no JWT Editor,
usada a função "Attack -> Embedded JWK" da extensão — que
automaticamente assina o token com a chave privada e embute a
chave pública no header. Acesso ao /admin concedido.

### Lab 5: JWT authentication bypass via jku header injection (Practitioner)
Servidor busca a chave pública em uma URL fornecida no campo
"jku" do header, sem validar se o domínio é confiável. Chave
pública própria (RSA) hospedada no Exploit Server do PortSwigger
em /jwks.json. Header editado manualmente: kid atualizado para
o kid da chave gerada, jku apontando para a URL do Exploit
Server. Token re-assinado com "Don't modify header". Dificuldade
encontrada: campo "File" do Exploit Server estava configurado
como "/exploit" por padrão — corrigido para "/jwks.json".

### Lab 6: JWT authentication bypass via kid header path traversal (Practitioner)
Servidor usa o campo "kid" do header para localizar o arquivo de
chave no sistema de arquivos. Sem sanitização do valor, é
possível usar path traversal para apontar para /dev/null (arquivo
vazio em sistemas Linux). Criada chave simétrica com valor
"k": "AA==" (base64 de string vazia). Header editado com
kid=../../../../../../../dev/null. Token assinado com essa chave
vazia — servidor lê /dev/null, obtém string vazia, e a assinatura
bate.

### Lab 7: JWT authentication bypass via algorithm confusion (Expert)
Servidor usa RSA (assimétrico) mas aceita HS256 (simétrico) por
implementação falha. Chave pública obtida via endpoint /jwks.json.
JWK importado no JWT Editor → convertido para PEM → PEM
codificado em base64 → usado como valor "k" em chave simétrica.
Header editado para alg: HS256, payload para sub: administrator.
Token re-assinado com HS256 usando a chave pública RSA como
segredo — servidor verifica com a mesma chave pública, assinatura
bate.

### Lab 8: JWT authentication bypass via algorithm confusion with no exposed key (Expert)
Variação do Lab 7 sem chave pública exposta. Dois tokens JWT
capturados de sessões diferentes. Chave pública derivada
matematicamente usando Docker:

    sudo docker run --rm -it portswigger/sig2n <token1> <token2>

Ferramenta gerou 2 candidatos de chave (x509 e pkcs1). Cada
token forjado testado no Repeater — o primeiro (x509, multiplier 1)
retornou 200 OK, confirmando a chave correta. Mesma técnica do
Lab 7: chave x509 base64 usada como "k" em chave simétrica,
alg trocado para HS256, sub para administrator, re-assinado com
"Don't modify header".

---

## Conclusão geral
Doze laboratórios completos cobrindo os principais vetores de
SSRF (ataque direto, ataque a sistema interno, bypass de
blacklist, bypass de whitelist) e os oito cenários de falha de
verificação de JWT (ausência de verificação, alg: none, chave
fraca, jwk injection, jku injection, kid path traversal, e
algorithm confusion com e sem chave exposta).

Recomendações gerais:
- SSRF: nunca confiar em requisições vindas do próprio servidor
  como automaticamente "confiáveis"; validar destino de URLs com
  parsing robusto de RFC 3986, não comparação de string simples
- JWT: sempre usar a função de verificação completa da biblioteca
  (nunca apenas decode); rejeitar explicitamente alg: none;
  usar allowlist de algoritmos aceitos; não confiar em kid, jku
  ou jwk fornecidos pelo cliente sem validação rigorosa; usar
  chaves secretas com entropia suficiente (256+ bits)
