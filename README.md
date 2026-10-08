# eleicao

Variação de votos para Presidente, por seção eleitoral e por local de votação, entre o 1º e o 2º turno de 2022 e o 1º turno de 2026: Lula (13), Bolsonaro (22), demais candidatos, brancos, nulos e abstenção. O foco é encontrar padrões, por exemplo se a mudança de local de uma seção afetou a votação, e cruzar com o perfil do eleitorado e dados do IBGE.

Os dados vêm do [Portal de Dados Abertos do TSE](https://dadosabertos.tse.jus.br) e do IBGE. As decisões de arquitetura estão em [docs/adr/](docs/adr/), e os dicionários de dados do TSE (`leiame`), em [docs/fontes/](docs/fontes/).

## Eleição 2026

Os resultados de 2026 vêm por ora do boletim de urna (`bweb_1t_<UF>_…`).

## O banco

As tabelas são separadas por granularidade e ligadas por `secao_id = cd_municipio * 10^8 + nr_zona * 10^5 + nr_secao`, uma chave estável entre execuções e entre anos.

| Tabela | Uma linha por | Conteúdo |
|---|---|---|
| `pleito` | pleito | `22_1t`, `22_2t`, `26_1t` |
| `municipio` | município | UF, nome, `cd_ibge`, `populacao_2026` |
| `secao` | seção | município, zona, número |
| `secao_pleito` | seção × pleito | tipo (principal, agregada ou distribuída), local, coordenadas, aptos, comparecimento, abstenções, `qt_lula`, `qt_bolsonaro`, `qt_outros`, `qt_brancos`, `qt_nulos` |
| `local_pleito` | local × pleito | soma das seções do local |
| `perfil_secao` | seção × ano do cadastro | eleitores por gênero, faixa etária, escolaridade e estado civil, com as seções agregadas somadas na principal |

### Views de variação

| View | Conteúdo |
|---|---|
| `v_secao_variacao` | toda seção principal em cada comparação (`22_1t-22_2t`, `22_2t-26_1t`, `22_1t-26_1t`), com a distância entre os locais (zero quando o local tem o mesmo nome, está a menos de 10 m, ou está a até 100 m com nome parecido) |
| `v_secoes_deslocadas` | as seções que mudaram de local mais de 1 km |
| `v_locais_deslocados` | seções que foram juntas do mesmo local para o mesmo local, somadas |
| `v_variacao_22_1t_26_1t`, `v_variacao_22_2t_26_1t` | uma linha por seção principal: distância, `aptos` (de 2026) e comparecimento e votos de cada pleito em % dos aptos daquele pleito (`_1`, `_2`) |
| `v_secao_var_pp_22_1t_26_1t`, `v_secao_var_pp_22_2t_26_1t` | seções principais que mudaram de local (`distancia_m > 0`): distância e `delta_<g>_pp` de cada grupo |

Para cada grupo (`lula`, `bolsonaro`, `outros`, `brancos`, `nulos`, `abstencao`) há duas colunas:

- `delta_<g>_abs`: variação em votos (na abstenção, em eleitores).
- `delta_<g>_pp`: variação em pontos percentuais dos aptos de cada pleito. Os seis grupos somam 100% dos aptos, então as seis variações de uma seção somam zero.

Exemplo: seções que mudaram de local entre os turnos de 2022, com a variação de Lula e Bolsonaro.

```python
import duckdb

con = duckdb.connect("eleicoes.duckdb", read_only=True)
con.sql("""
    SELECT nm_municipio, nr_zona, nr_secao, local_1, local_2, distancia_m,
           round(delta_lula_pp, 1) AS delta_lula, round(delta_bolsonaro_pp, 1) AS delta_bolsonaro
    FROM v_secoes_deslocadas
    WHERE mudanca = '22_1t-22_2t'
""").show()
```

## Cuidados

- **Local de 2022:** o cadastro de locais de 2022 foi regerado pelo TSE em 2024 e diverge do resultado da eleição em cerca de 1,4% das seções. Por isso, o local de cada seção vem do arquivo de votos, e o cadastro entra só para coordenadas e bairro. Seções com local divergente e sem coordenada ficam sem distância (coluna `local_divergente`).
- **Coordenadas fora do município:** cada coordenada é testada contra a malha municipal do IBGE. Se está a mais de 1 km fora do próprio município, `coord_valida` é falso e a distância da comparação fica nula (a seção sai das deslocadas), a não ser que o local tenha o mesmo nome nas duas pontas.
- **Mesma seção entre anos:** a mesma chave em 2022 e 2026 não garante o mesmo eleitorado, porque o TSE renumera e agrega seções. A comparação por local é mais robusta.
- **Seções pequenas:** com poucos comparecentes, a variação em pp é ruidosa. Filtre por `comparecimento_1`.
- **2026 pelo boletim de urna:** só há urnas apuradas; 42 seções do exterior sem boletim ficam com votos nulos. O número 22 (`qt_bolsonaro`) é Flávio Bolsonaro em 2026.
- **Local de 2026:** o boletim traz só o número do local; nome, endereço e coordenadas vêm do cadastro de 2026 (anterior à eleição). Em ~6.600 seções o local do boletim difere do cadastro, e na maioria delas o local nem existe no cadastro: ficam sem nome, sem coordenada e sem distância.
- **Fora do banco:** raça/cor e identidade de gênero (não coletadas em 2022) e o PIB municipal. A planilha de PIB de 2023 só tem rankings e fica apenas no Parquet.
