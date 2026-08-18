# Writeup: Reflected XSS - DVWA - Metasploitable2

## Ambiente
- Atacante: Kali Linux (VM VirtualBox)
- Alvo: DVWA (Damn Vulnerable Web App) rodando em Metasploitable2
- URL: http://192.168.56.10/dvwa/vulnerabilities/xss_r/
- Nível de segurança: Low

## 1. Identificação da vulnerabilidade
Campo "name" da página reflete a entrada do usuário diretamente
no HTML da resposta, sem sanitização ou escape de caracteres
especiais.

## 2. Prova de conceito
Payload de teste:

    <script>alert('XSS')</script>

Resultado: pop-up de alerta do navegador exibido, confirmando
que o script injetado foi executado pelo navegador (não tratado
como texto).

## 3. Demonstração de impacto - roubo de cookie de sessão
Payload:

    <script>alert(document.cookie)</script>

Resultado: cookie de sessão exposto no pop-up:

    security=low; PHPSESSID=3eae270bc83acefd7c46b573fa5d9c11

## Impacto
Em um cenário real, o script poderia enviar o cookie capturado
para um servidor controlado pelo atacante (ao invés de exibir em
alert()), permitindo sequestro de sessão (session hijacking) —
o atacante poderia se autenticar como a vítima sem conhecer a
senha, usando apenas o cookie roubado.

## Conclusão
Reflected XSS presente por ausência de sanitização/escape de
entrada do usuário refletida na página. Como correção: aplicar
escape de caracteres HTML (htmlspecialchars ou equivalente) em
toda entrada refletida, implementar Content Security Policy
(CSP), e marcar cookies de sessão como HttpOnly (impedindo
acesso via JavaScript).
