# Uso de espressões aggregate e relação espacial no Qgis

O QGIS possui uma poderosa funcionalidade chamada aggregate que permite agrupar, resumir, totalizar valores e geometrias. O que o torna mais interessante, ao meu ver, é a possibilidade de acrescentar relações espaciais: **Equals, Disjoint, Intersects, Within, Touches, Crosses e Overlaps**. 

O uso mais comum dessa função é gerar estatísticas de um campo numérico de uma camada, por exemplo: somar todos os valores do campo de passageiros na camada estações ferroviárias.

Antes de dar o exemplo da questão acima, é importante demosntrar a sintaxe dessa função: 

```
aggregate(layer, aggregate, expression, [filter], [concatenator=’’], [order_by])
```
  * **layer:** uma string, representando um nome de camada ou um ID de camada
  * **aggregate:** uma string correspondente à agregação a ser calculada. As opões válidas são:
    - count
    - count_distinct
    - count_missing
    - min
    - max
    - sum
    - mean
    - median
    - stdev
    - stdevsample
    - range
    - minority
    - majority
    - q1: primeiro quartil
    - q3:terceiro quartil
    - iqr: intervalo interquartil
    - min_length: comprimento mínimo da string
    - max_length: comprimento máximo da string
    - concatenate: unir strings com um concatenador
    - concatenate_unique: junte strings únicas com um concatenador
    - collect: cria uma geometria multiparte agregada
    - array_agg: cria um array de valores agregados

  * **expression:** sub expressão ou nome do campo para agregar
  * **filter:** expressão de filtro opcional para limitar os recursos usados ​​para calcular a agregação. Campos e geometria são das feições na camada unida. O recurso de origem pode ser acessado com a variável @parent.
  * **concatenator:** optional string to use to join values for ‘concatenate’ and ‘concatenate_unique’ aggregates
  * **order_by:** expressão de filtro opcional para ordenar os recursos usados ​​para calcular o agregado. Campos e geometria são as das feições da camada unida. Por padrão, os recursos serão retornadas em uma ordem não especificada.

O Aggregate possui cláusulas, bem próximas às _query_ em **SQL**, de modo que podemos refinar as expressões, otimizando as consultas ou cálculos realizados. Entre essa cláusulas está:
**Group_by** que permite agrupar uma consulta por um campo categórico, **Filter** que permite refinar a consulta anterior e **Order_by**, que pode organizar os registros ou resultados obtidos levando em consideração os critérios emitidos pelo analista.

Já que abordamos a sintaxe, podemos trazer a expressão que soluciona o questionamento anterior: 

```
aggregate(layer:='estações_ferroviárias',aggregate:='sum',expression:="passageiros")
```
## Acrescentando relações espaciais
O que é mais legal em utilizar aggregate é a possibilidade de usar relações espacais nas consultas. Por exemplo: "Qual á área de intersecção entre unidades de conservação (UC) de gestão do IBRAM e áreas de proteção de manancial (APM)?" Neste caso, há APMs que fazem intersecção entre mais de uma UC, portanto é interessante mostrar qual unidade e qual a área de intersecção, ou seja, mostrar duas informações concatenadas: Nome da UC e Área (ha) de intersecção.
Pessando nessa problemática, criei a seguinte expressão: 

```
/*
Medir a área que faz intersecção entre as Áreas de Proteção de Manancial
*/
aggregate(
	'Unidades_de_Conserva__o_Gest_o_IBRAM_f52ffc88_d98a_4ecc_a0e6_408c9cd4381d',                                 
	'concatenate',                                           
	concat("sigla",
			to_string(': '),
			to_string(
						round(
							area(intersection(geometry(@parent),$geometry)),3)
							)
					),
	concatenator:=', ',                           
	filter:=intersects(geometry(@parent),$geometry)
)
```
Também é interessante mostrar a proporção em % da área de intersecção:
```
/*
Medir a área em (%) que faz intersecção entre as Áreas de Proteção de Manancial
*/
aggregate(
	'Unidades_de_Conserva__o_Gest_o_IBRAM_f52ffc88_d98a_4ecc_a0e6_408c9cd4381d',                                 
	'concatenate',                                           
	concat("sigla",
			to_string(': '),
			 round(area(intersection(geometry(@parent),$geometry))/area(geometry(@parent))*100,2),
			 to_string('%')
			)							,
	concatenator:=', ',                           
	filter:=intersects(geometry(@parent),$geometry)
)
```
Outra questão que pode surgir: Qual o volume das outorgas superficiais que fazem intersecção entre as APMs? 
Segue a solução: 

```
/*
Somar o volume (v_max) das outorgas superficiais que faz intersecção entre as Áreas de Proteção de Manancial
*/
aggregate(
	'Outorga_Capta__o_Superficial',                        
	'sum',
	"v_max",
	filter:=intersects(geometry(@parent),$geometry) 
)
```
Um outro exemplo que também vale a pena ser ilustado é: Qual a soma das áreas dos imoveis rurais cadastrados pela SEAGRE com status 'Contrato Assinado' faz intersecção entre as AMPs? 
O bacana dessa solução é que além de usar a relação espacial, foi realizado um refinamento no filtro.
```
/*
a10
Somar a dos imoveis rurais cadastrados pela SEAGRI que faz intersecção entre as Áreas de Proteção de Manancial
Status -  'CONTRATO ASSINADO'
*/
aggregate(
	'processos_seagri___ativos',                        
	'sum',
	round(area(intersection(geometry(@parent),$geometry)),3),
	filter:=intersects(geometry(@parent),$geometry) 
	and "status" like 'CONTRATO ASSINADO' 
)
```

Portanto, esses são exemplos de vários que surgem no dia-a-dia de um analista em geoprocessamento e que de forma simples pode ser respondida e preenchida em uma tabela de atrinutos de um arquivo vetorial!


