# Writeup: DVWA - CSRF, Stored XSS e Brute Force - Metasploitable2

## Ambiente
- Atacante: Kali Linux (VM VirtualBox)
- Alvo: DVWA (Damn Vulnerable Web App) rodando em Metasploitable2
- Nível de segurança: Low

---

## 1. CSRF (Cross-Site Request Forgery)

**URL:** http://192.168.56.10/dvwa/vulnerabilities/csrf/

### Identificação
A funcionalidade de troca de senha aceita todos os parâmetros
necessários via requisição GET, sem token anti-CSRF nem
verificação de origem da requisição:

    ?password_new=NOVASENHA&password_conf=NOVASENHA&Change=Change

### Construção do ataque
Página HTML maliciosa hospedada localmente, contendo um link
disfarçado que aponta para a URL de troca de senha:

    <a href="http://192.168.56.10/dvwa/vulnerabilities/csrf/?password_new=hackeado123&password_conf=hackeado123&Change=Change#">Clique aqui</a>

### Execução
Com a vítima autenticada no DVWA, o clique no link malicioso
disparou a troca de senha sem qualquer confirmação adicional.
Confirmado via login bem-sucedido posterior com a nova senha
"hackeado123".

### Observação técnica
Uma tentativa inicial usando uma tag `<img>` carregando a URL
silenciosamente falhou devido à proteção SameSite Cookie dos
navegadores modernos. A técnica funcionou ao simular uma
navegação de topo (clique em link), que o SameSite=Lax ainda
permite.

### Correção recomendada
Implementar token CSRF único por sessão/formulário, validar o
cabeçalho Referer/Origin, e preferir requisições POST a GET para
ações que alteram estado.

---

## 2. Stored XSS

**URL:** http://192.168.56.10/dvwa/vulnerabilities/xss_s/

### Identificação
O campo "Message" do livro de visitas (guestbook) armazena a
entrada do usuário sem sanitização e a exibe para todos os
visitantes da página.

### Prova de conceito
Payload inserido no campo Message:

    <script>alert('XSS Stored')</script>

Resultado: o pop-up de alerta passou a ser exibido
automaticamente toda vez que a página era recarregada, mesmo sem
reenvio do formulário — confirmando persistência no banco de
dados.

### Impacto
Diferente do XSS Reflected, o Stored afeta automaticamente
qualquer usuário que simplesmente visite a página do guestbook,
tornando o alcance do ataque muito maior.

### Correção recomendada
Aplicar escape de caracteres HTML tanto na entrada quanto na
exibição, implementar Content Security Policy (CSP), e
validar/filtrar conteúdo antes de persistir no banco de dados.

---

## 3. Brute Force (ausência de rate limiting)

**URL:** http://192.168.56.10/dvwa/vulnerabilities/brute/

### Identificação
O formulário de login não implementa limite de tentativas,
CAPTCHA, ou bloqueio temporário de conta após múltiplas
tentativas falhas.

### Ataque automatizado
Uso do Hydra para testar sistematicamente senhas contra o
usuário admin:

    hydra -l admin -P /usr/share/wordlists/rockyou.txt 192.168.56.10 \
      http-get-form "/dvwa/vulnerabilities/brute/:username=^USER^&password=^PASS^&Login=Login:H=Cookie\: security=low; PHPSESSID=<sessao>:F=Username and/or password incorrect."

Resultado: a ferramenta executou milhares de tentativas
consecutivas sem qualquer bloqueio, captcha ou atraso imposto
pela aplicação, confirmando a ausência de proteção contra força
bruta.

### Correção recomendada
Implementar bloqueio temporário de conta após N tentativas
falhas, CAPTCHA após tentativas suspeitas, e
monitoramento/alerta de tentativas de login em massa.
