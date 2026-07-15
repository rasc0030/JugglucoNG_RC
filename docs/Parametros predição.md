## Apêndice: Teoria e Prática dos Parâmetros Preditivos

Para que a Simulação Preditiva e a Calculadora de Dose funcionem com precisão, o aplicativo precisa de parâmetros que traduzam o metabolismo único do seu corpo para a matemática. Configurações incorretas geram gráficos irreais (como picos exagerados ou quedas falsas).

Abaixo, explicamos a teoria por trás de cada variável e apresentamos métodos práticos para você estimar e ajustar seus próprios valores.

## 1. Fator de Sensibilidade à Insulina (FSI / ISF)

**Teoria:** Representa quantos mg/dL a sua glicose cai quando você aplica 1 Unidade (1 U) de insulina rápida. É a "força" da insulina no seu corpo. Se a sua sensibilidade é 50 mg/dL/U, aplicar 1 U fará uma glicose de 150 cair para 100.

**Como estimar na prática (A Regra dos 1800):**
A medicina utiliza uma regra geral baseada na sua Dose Total Diária (DTD) de insulina (a soma de toda a insulina basal + rápida que você toma em um dia).

* **Fórmula:** `1800 ÷ Dose Total Diária (DTD)`
* **Exemplo:** Se você usa 20U de basal e 16U de rápida por dia, sua DTD é 36U. `1800 ÷ 36 = 50`. Sua sensibilidade estimada é de 50 mg/dL por Unidade.

**Como testar e calibrar:**

1. Comece com a glicose estável e um pouco alta (ex: 180 mg/dL), de preferência em jejum e sem insulina ativa (IOB zerado).
2. Aplique 1 U de insulina rápida (ou uma dose pequena e segura que você conheça).
3. Aguarde o tempo total de ação da insulina (4 a 5 horas) sem comer.
4. Subtraia a glicose final da glicose inicial. A diferença é a sua Sensibilidade real.

## 2. Razão de Carboidrato (Relação C/I)

**Teoria:** Representa quantos gramas de carboidrato são totalmente "cobertos" ou neutralizados por 1 Unidade de insulina rápida. Se a sua razão é 15g/U, significa que para cada 15 gramas de carbo que você comer, precisará de 1 U de insulina para que a sua glicose termine no mesmo valor em que começou.

**Como estimar na prática (A Regra dos 500):**
Também se baseia na Dose Total Diária (DTD) de insulina.

* **Fórmula:** `500 ÷ Dose Total Diária (DTD)`
* **Exemplo:** Com a mesma DTD de 36U. `500 ÷ 36 = 13.8` (arredondado para 14). Sua Razão Carbo estimada é de 14g/U.

**Como testar e calibrar:**

1. Comece com a glicose estável no alvo (ex: 100 mg/dL) e sem insulina ativa.
2. Faça uma refeição contendo um valor exato e conhecido de carboidratos (ex: 45g de pão ou suco).
3. Aplique a insulina calculada pela sua razão atual (se a razão for 15g/U, aplique 3 U).
4. Aguarde o fim da absorção e da insulina (umas 4 horas). Se a glicose voltou para 100, sua razão está correta. Se terminou alta, a razão está muito frouxa (você precisa de um número menor, como 12g/U). Se teve hipoglicemia, a razão está muito forte (precisa de um número maior, como 18g/U).

## 3. Taxa de Absorção de Carboidratos (g/h)

**Teoria:** Define a velocidade padrão em que o sistema assume que o estômago esvazia os carboidratos na corrente sanguínea. Uma taxa de 35g/h significa que uma refeição de 70g levará 2 horas inteiras para ser absorvida.

**Como estimar na prática:**
Não há fórmula matemática universal, pois a digestão depende de fatores como gastroparesia, índice glicêmico e flora intestinal. O padrão da maioria dos aplicativos (como o AndroidAPS) fica entre **20g/h e 40g/h**.

**Como testar e calibrar (Usando o Gráfico do JugglucoNG):**

1. Faça uma refeição mista comum.
2. Deixe a velocidade padrão no aplicativo (sem selecionar botões rápidos ou lentos).
3. Compare a **Curva Preditiva (Simulação)** com a **Curva Real (Sensor)** nas 2 horas seguintes.
   * **Pico preditivo exagerado:** Se a simulação subiu rápido demais, mas sua glicemia real subiu lentamente, o app acha que você absorve mais rápido do que a realidade. *Reduza* a taxa padrão (ex: de 35g/h para 20g/h).
   * **Pico preditivo atrasado:** Se a simulação mostrou uma subida suave, mas sua glicose real disparou rápido, o app acha que você absorve lentamente. *Aumente* a taxa (ex: de 30g/h para 45g/h).

## 4. Tempo de Ação da Insulina (Perfil Farmacológico / IOB)

**Teoria:** É o tempo total que a insulina rápida leva para começar a agir (*Onset*), atingir o pico (*Peak*) e sair completamente do seu corpo (*Duration*). Essa informação dita a curva com que o aplicativo abate a "Insulina Ativa" (IOB).

**Como estimar na prática:**
Geralmente usa-se a bula da própria insulina como ponto de partida (ex: Fiasp age mais rápido que Novolog/Lispro). O padrão comum para insulina rápida (Lispro/Asparte) é uma duração de **4 a 5 horas**.

**Como testar e calibrar (Cuidado com a "Cauda"):**
O maior indicador de que o tempo de ação está configurado errado é o que acontece nas últimas horas da curva (a cauda).

1. Corrija uma glicose alta sem comer nada.
2. Na marca de 4 horas, verifique sua glicose. Se o aplicativo já estiver informando que o IOB zerou (a insulina "acabou"), mas **sua glicose continuar caindo por mais 1 a 2 horas**, a duração da insulina no seu corpo é maior do que a configurada no aplicativo.
3. *Solução:* Você deve aumentar o tempo de duração daquele perfil de insulina nas configurações (ex: de 4 horas para 5 ou 6 horas). Se não fizer isso, o app pode induzi-lo a aplicar uma correção perigosa porque achou que o seu corpo já estava "limpo" da dose anterior.
