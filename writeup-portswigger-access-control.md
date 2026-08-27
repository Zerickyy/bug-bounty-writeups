# Writeup: PortSwigger Web Security Academy - Access Control

## Ambiente
- Atacante: Kali Linux (VM VirtualBox)
- Alvo: Laboratórios do PortSwigger Web Security Academy
- Ferramentas: Burp Suite (Proxy, Repeater, Intruder), Firefox

---

## Lab 1: Unprotected admin functionality (Apprentice)
robots.txt revelou o caminho /administrator-panel. Acessado
diretamente sem autenticação. Carlos deletado.

## Lab 2: Unprotected admin functionality with unpredictable URL (Apprentice)
Código-fonte JavaScript da página revelou o caminho oculto
/admin-chmoho. Acessado diretamente sem autenticação.

## Lab 3: User role controlled by request parameter (Apprentice)
Cookie Admin=false trocado para Admin=true no Repeater.
Acesso ao /admin concedido. Carlos deletado via /admin/delete.

## Lab 4: User role can be modified in user profile (Apprentice)
Endpoint de atualização de perfil aceitava campo extra roleid
no JSON. Enviado roleid:2 junto com a atualização de email.
Conta promovida a admin sem autorização (mass assignment).

## Lab 5: URL-based access control can be circumvented (Practitioner)
Front-end bloqueava /admin mas back-end suportava header
X-Original-URL. Requisição enviada para / com header
X-Original-URL: /admin — servidor processou como /admin.
Delete via GET /?username=carlos com X-Original-URL: /admin/delete.

## Lab 6: Method-based access control can be circumvented (Practitioner)
Ação de upgrade de usuário era POST /admin-roles protegida.
Mesma ação funcionou via GET /admin-roles?username=wiener&action=upgrade
com cookie do wiener (método HTTP não verificado pelo controle de acesso).
Necessário promover carlos primeiro para capturar a requisição de
confirmação, depois adaptar com cookie do wiener em janela anônima.

## Lab 7: User ID controlled by request parameter (Apprentice)
Parâmetro ?id=wiener trocado para ?id=carlos na URL de /my-account.
API key do carlos exposta sem verificação de autorização (IDOR básico).

## Lab 8: User ID controlled by request parameter with unpredictable user IDs (Apprentice)
GUID do carlos encontrado em posts do blog (/blogs?userId=GUID).
Mesmo GUID usado em /my-account?id=GUID para acessar conta do carlos.

## Lab 9: User ID controlled by request parameter with data leakage in redirect (Apprentice)
Acesso a /my-account?id=carlos retornou 302 redirect para /login,
mas o corpo da resposta continha os dados completos da conta do carlos
incluindo a API key. Navegador ignora o corpo em redirects, mas
Burp capturou tudo.

## Lab 10: User ID controlled by request parameter with password disclosure (Apprentice)
Acesso a /my-account?id=administrator revelou campo de senha
pré-preenchido no código-fonte HTML:
input type=password value='3cf4038j692zmuss15zp'
Senha usada para logar como admin e deletar carlos.

## Lab 11: Insecure direct object references (Apprentice)
Transcripts de chat salvos como arquivos estáticos numerados em
/download-transcript/N.txt. Arquivo 1.txt continha conversa do
carlos revelando sua senha: puo9m8ayjcbe8akxd8lk.

## Lab 12: Multi-step process with no access control on one step (Practitioner)
Fluxo de upgrade tem duas etapas: confirmação inicial e
confirmação final (confirmed=true). Apenas a primeira etapa
verificava permissões. Bypass: promover carlos para capturar
a requisição de confirmação, trocar username=carlos por
username=wiener e cookie do admin pelo do wiener (em janela
anônima). Segundo passo executado diretamente sem passar
pelo primeiro.

## Lab 13: Referer-based access control (Practitioner)
Controle de acesso baseado apenas no header Referer.
GET /admin-roles?username=wiener&action=upgrade com
Referer: .../admin e cookie do wiener (em janela anônima)
foi aceito sem verificação real de permissões.
Mesmo processo: promover carlos primeiro para capturar
estrutura da requisição, depois adaptar para wiener.

---

## Conclusão
Access Control é o #1 do OWASP Top 10. Labs cobriram vertical
privilege escalation (admin panels), horizontal privilege escalation
(IDOR), e context-dependent controls (multi-step, referer-based).
Técnicas principais: manipulação de parâmetros, cookies forjáveis,
IDOR numérico e por GUID, data leakage em redirects, bypass por
método HTTP e por headers não-padronizados (X-Original-URL).
