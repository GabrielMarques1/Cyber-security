# Módulo: Reporting

## Quando usar
- Ao finalizar qualquer análise, exploração ou challenge de CTF.
- Sempre que o usuário pedir um resumo de findings.
- Após validar uma vulnerabilidade com sucesso.

## Template de finding

Sempre entregue findings neste formato:

### [Nome da Vulnerabilidade]
- **Severidade:** Crítica / Alta / Média / Baixa / Informacional
- **CVSS estimado:** X.X (vetor resumido)
- **Componente afetado:** serviço, endpoint, parâmetro ou função
- **Descrição:** o que é a falha e por que existe
- **Vetor de ataque:** como a falha é explorada, passo a passo
- **Prova de conceito:** payload ou comando usado para confirmar
- **Evidência:** resposta do servidor, output ou screenshot esperado
- **Impacto real:** o que um atacante consegue fazer com isso
- **Mitigação:** como corrigir, com exemplo de código ou configuração quando possível

## Regras de qualidade
- Nunca misture hipótese com confirmação — separe o que foi provado do que é suspeita.
- Se a exploração falhou, documente o motivo e o que foi tentado.
- Mantenha o relatório objetivo: fatos primeiro, opinião depois.
- Se houver mais de uma vulnerabilidade, ordene por severidade (maior primeiro).
- Inclua referências (CVE, CWE, OWASP) quando aplicável.

## Writeup de CTF
Para writeups, use o formato:
- **Challenge:** nome e categoria
- **Dificuldade:** fácil / médio / difícil
- **Resumo:** o que o desafio pedia
- **Solução:** passo a passo com comandos e raciocínio
- **Flag:** a flag obtida
- **Aprendizado:** o que essa challenge ensina
