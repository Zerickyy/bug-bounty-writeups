# Writeup: PortSwigger Web Security Academy - Business Logic Vulnerabilities

## Ambiente
- Atacante: Kali Linux (VM VirtualBox)
- Alvo: Laboratórios do PortSwigger Web Security Academy
- Ferramentas: Burp Suite (Proxy, Repeater, Intruder), Firefox

---

## Lab 1: Excessive trust in client-side controls (Apprentice)
Requisição POST /cart incluía o campo price=133700 no corpo.
Servidor aceitava o preço enviado pelo cliente sem validar
no servidor. Preço alterado para price=1 no Repeater.
Jaqueta de $1337 comprada por $0.01.

## Lab 2: High-level logic vulnerability (Apprentice)
Servidor validava o preço mas não validava se a quantidade
podia ser negativa. Adicionada quantidade negativa de produto
barato (AbZorba Ball, quantity=-26) para subtrair valor do
carrinho e zerar o total. Jaqueta comprada com crédito disponível.

## Lab 3: Inconsistent handling of exceptional input (Practitioner)
Aplicação truncava emails longos em 255 caracteres. Registro
com email de 238 'a's + @dontwannacry.com + sufixo do exploit
server fazia o servidor truncar nos 255 chars, registrando o
email como aaa...@dontwannacry.com (domínio autorizado para admin).
Confirmação chegou no exploit server via subdomínio.
Acesso ao painel admin obtido. Carlos deletado.

## Lab 4: Inconsistent security controls (Apprentice)
Aplicação restringia admin a emails @dontwannacry.com mas
permitia trocar o email após o registro sem reverificar o
domínio. Registrou conta normal, depois trocou email para
qualquercoisa@dontwannacry.com. Acesso ao admin concedido.

## Lab 5: Weak isolation on dual-use endpoint (Practitioner)
Endpoint POST /my-account/change-password verificava
current-password apenas quando o username era o mesmo do
usuário logado. Removendo o campo current-password e trocando
username=wiener para username=administrator, a senha do admin
foi alterada sem conhecer a senha atual.

## Lab 6: Password reset broken logic (Apprentice)
Token de reset de senha não estava vinculado ao username.
Fluxo: solicitou reset para wiener, recebeu token por email,
acessou o link de reset, interceptou o POST de confirmação
e trocou username=wiener para username=carlos. Senha do
carlos alterada sem acesso ao email dele.

---

## Labs pulados/pendentes
- Low-level logic flaw (Practitioner): integer overflow via
  adição massiva de itens — conceito compreendido, lab
  requer muitas iterações com Burp Intruder
- Infinite money logic flaw (Practitioner): ciclo gift card
  + cupom SIGNUP30 gera $3 de lucro por ciclo — conceito
  validado manualmente, automação via Burp Macro em progresso

---

## Conclusão
Business Logic vulnerabilities são invisíveis para scanners
automáticos — exigem raciocínio sobre o fluxo de negócio.
Técnicas cobertas: manipulação de preço client-side, quantidade
negativa, truncamento de input, controles inconsistentes entre
registro e uso, endpoints de uso duplo com isolamento fraco,
e tokens de reset não vinculados ao usuário.
