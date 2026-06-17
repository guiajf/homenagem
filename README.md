# Mapeamento e análise de vias ubanas com OSMnx

### Introdução

Em Três Corações/MG, há duas vias urbanas em homenagem ao seu filho mais ilustre, além de uma praça controversa: a *Avenida Rei Pelé* e a *Rua Edson Arantes do Nascimento*, onde está situada a [*Casa Pelé*](https://ge.globo.com/mg/sul-de-minas/futebol/noticia/2022/12/29/casa-pele-resgata-memoria-e-mistica-do-local-em-que-o-rei-viveu-ate-os-tres-anos-no-sul-de-minas.ghtml), reconstituição histórica do local onde o craque nasceu. Uma homenageia o mito; a outra, o homem.

Neste projeto, desenvolvemos um código em **Python**, com o auxílio do pacote **Osmnx**, para mensurar a extensão e identificar outros atributos das duas vias, assim como destacar os traçados delas em um mapa interativo no **Folium**.

Em primeiro lugar extraimos todos os elementos viários da região de interesse utilizando a função *ox.features_from_place()*. Depois, convertemos os dados para o sistema de projeção específico para a região do Brasil. Em seguida, realizamos uma busca para retornar os dados necessários para realizar a visualização geográfica e efetuar cálculos métricos. Finalmente, unificamos os segmentos das vias e definimos os padrões gráficos.

### Objetivo

O objetivo consiste em extrair, processar e visualizar todos os segmentos de uma via específica a partir de dados do **OpenStreetMap**, incluisive o tratamento para pistas duplas, segmentos desconectados e projeção precisa para cálculos métricos.

O código desenvolvido foi desenvolvido para identificar todos os trechos dessas vias, unificá-los visualmente e destacar pontos de interesse, como a *Casa Pelé*.

### Bibliotecas

Carregamos as seguintes bibliotecas:





- **osmnx**: biblioteca especializada para modelagem de redes urbanas
a partir de dados do *OpenStreetMap*. Permite baixar grafos viários
reais e calcular rotas. Utilizada para obter a rede de ruas de
Juiz de Fora (raio de 30 km) e calcular caminhos mais curtos.

- **folium**: biblioteca para visualização geoespacial interativa,
baseada em *Leaflet.js*. Plugins *Fullscreen* e *MeasureControl*
adicionam tela cheia e ferramenta de medição ao mapa.

- **pandas**: biblioteca fundamental para análise de dados em Python,
oferece estruturas como *DataFrame* e *Series* para manipulação e
análise de dados tabulares. 

- **numpy**: pacote essencial para computação científica, fornece
suporte a *arrays* multidimensionais e funções matemáticas de
alto desempenho.

- **shapely.geometry**: módulo da biblioteca *Shapely* especializado na definição e manipulação de objetos geométricos fundamentais. Fornece classes como *LineString* (para representar sequências de pontos conectados) e *MultiLineString* (para representar coleções de linhas). Utilizado neste projeto para trabalhar com as geometrias das vias extraídas do OpenStreetMap e identificar a estrutura de cada segmento.

- **shapely.ops**: módulo da biblioteca *Shapely* que implementa operações geométricas avançadas. A função *linemerge* é utilizada para tentar unir múltiplos segmentos de uma mesma via em uma linha contínua, identificando a via principal e tratando casos onde os trechos estão desconectados ou fragmentados.

- **geopandas**: biblioteca que estende o pandas para manipulação de dados geoespaciais. Combina as capacidades do *pandas* com as funcionalidades geométricas do *Shapely*, permitindo armazenar geometrias, calcular áreas, comprimentos e realizar operações espaciais. Neste projeto, é utilizada para manipular os dados vetoriais baixados do *OpenStreetMap*, aplicar projeções **UTM** e acessar as propriedades geométricas dos segmentos viários.

- **warnings**: módulo da biblioteca padrão para controle de mensagens de aviso. Utilizado para suprimir alertas técnicos e manter a saída limpa e focada nos resultados.


```python
import osmnx as ox
import folium
import pandas as pd
import numpy as np
from shapely.geometry import LineString, MultiLineString
from shapely.ops import linemerge
from folium.plugins import MeasureControl, Fullscreen
import geopandas as gpd
import warnings
warnings.filterwarnings('ignore')
```

### Definimos os limites geográficos


```python
place = "Três Corações, Minas Gerais, Brazil"
```

### Extraímos dados espaciais

A primeira etapa consiste em obter todos os elementos viários da região de interesse utilizando a função *ox.features_from_place()*. Esta função retorna todos os elementos geométricos (linhas, polígonos, pontos) que possuem a tag *highway*, garantindo a captura completa da infraestrutura viária.


```python
gdf = ox.features_from_place(place, tags={"highway": True})
```

### Convertermos para a projeção UTM

Os dados brutos do **OpenStreetMap** estão em coordenadas geográficas (*WGS84 - EPSG:4326*), que não são ideais para cálculos de distância devido às distorções da curvatura terrestre. Para resolver esse problema, convertemos os dados para o sistema de projeção *UTM - EPSG:31983* (*SIRGAS 2000*, Zona 23S), específico para a região do Brasil. Essa projeção minimiza distorções e permite cálculos precisos de comprimento em metros.


```python
gdf_utm = gdf.to_crs("EPSG:31983")
```

### Função para encontrar todos os segmentos de uma via

Definimos a função *encontrar_todos_segmentos()*, que implementa uma busca inteligente baseada em expressões regulares parciais nos nomes das vias, e retorna dois objetos:
*segmentos originais*, para visualização geográfica e *segmentos projetados*, para cálculos métricos precisos.


```python
def encontrar_todos_segmentos(gdf_original, gdf_utm, nomes_possiveis):
    """
    Encontra TODOS os segmentos de uma via, incluindo pistas duplas
    e segmentos desconectados.
    """
    mascara = pd.Series(False, index=gdf_original.index)
    
    for nome in nomes_possiveis:
        mascara |= gdf_original["name"].str.contains(nome, case=False, na=False)
    
    segmentos_original = gdf_original[mascara].copy()
    
    if segmentos_original.empty:
        print(f" Nenhum segmento encontrado para: {nomes_possiveis}")
        return None
    
    # Pega os mesmos índices na projeção UTM
    segmentos_utm = gdf_utm.loc[segmentos_original.index].copy()
    
    # Calcula comprimentos com projeção correta
    comprimentos = segmentos_utm.geometry.length  # em metros (já na projeção)
    
    print(f"\n Encontrados {len(segmentos_original)} segmentos para '{nomes_possiveis[0]}'")
    
    # Analisa os tipos de geometria
    geom_types = segmentos_original.geometry.geom_type.value_counts()
    print("    Tipos de geometria:")
    for geom_type, count in geom_types.items():
        print(f"      - {geom_type}: {count}")
    
    print(f"    Comprimento total: {comprimentos.sum():.0f} metros")
    print(f"    Comprimento médio: {comprimentos.mean():.0f} metros")
    print(f"    Número total de trechos: {len(segmentos_original)}")
    
    return segmentos_original, segmentos_utm
```

### Localizamos todos os segmentos da *Avenida Rei Pelé*


```python
segmentos_rei_orig, segmentos_rei_utm = encontrar_todos_segmentos(
    gdf, gdf_utm,
    [
        "Avenida Rei Pelé",
        "Av. Rei Pelé",
        "Avenida Deputado Renato Azeredo",
        "Av. Deputado Renato Azeredo",
        "Rei Pelé"
    ]
)
```

    
     Encontrados 74 segmentos para 'Avenida Rei Pelé'
        Tipos de geometria:
          - LineString: 74
        Comprimento total: 11817 metros
        Comprimento médio: 160 metros
        Número total de trechos: 74


### Localizamos todos os segmentos da *Rua Edson Arantes*


```python
segmentos_edson_orig, segmentos_edson_utm = encontrar_todos_segmentos(
    gdf, gdf_utm,
    [
        "Rua Edson Arantes do Nascimento",
        "Edson Arantes",
        "Rua Edson Arantes"
    ]
)
```

    
     Encontrados 5 segmentos para 'Rua Edson Arantes do Nascimento'
        Tipos de geometria:
          - LineString: 5
        Comprimento total: 576 metros
        Comprimento médio: 115 metros
        Número total de trechos: 5


### Criamos o mapa


```python
mapa = folium.Map(location=[-21.697, -45.253], zoom_start=15)
```

### Definimos uma função para adicionar as vias no mapa

A função *linemerge()*, da biblioteca **Shapely**, une os segmentos individuais em uma linha contínua, identificando a via principal. Ambas as vias apresentaram apenas geometrias do tipo *LineString*, indicando que não há polígonos ou pontos isolados associados a essas ruas no **OpenStreetMap**. 


```python
def adicionar_via_completa(segmentos_orig, segmentos_utm, nome_exibicao, cor_principal):
    """Adiciona TODOS os segmentos da via ao mapa com destaque"""
    
    if segmentos_orig is None or segmentos_orig.empty:
        print(f" Via '{nome_exibicao}' não encontrada.")
        return
    

    try:
        # Tenta unir todos os segmentos em uma única linha
        todas_geometrias = list(segmentos_orig.geometry)
        linha_unida = linemerge(todas_geometrias)
        
        if linha_unida.geom_type == "MultiLineString":
            # Pega a linha mais longa (via principal)
            linha_principal = max(linha_unida.geoms, key=lambda x: x.length)
            print(f"   📍 Via principal identificada (mais longa)")
        else:
            linha_principal = linha_unida
        
        # Desenha a via principal em destaque
        if linha_principal.geom_type == "LineString":
            coords = list(linha_principal.coords)
            folium.PolyLine(
                locations=[(lat, lon) for lon, lat in coords],
                color=cor_principal,
                weight=10,
                opacity=0.8,
                popup=f"{nome_exibicao} (VIA PRINCIPAL)",
                tooltip=f"{nome_exibicao}"
            ).add_to(mapa)
            
            
    except Exception as e:
        print(f"   ⚠️ Erro ao unir segmentos: {e}")
    
    # ---  TODOS OS SEGMENTOS INDIVIDUAIS ---
    print(f"    Desenhando {len(segmentos_orig)} segmentos individuais...")
    
    for idx, row in segmentos_orig.iterrows():
        geom = row.geometry
        
        # Extrai coordenadas
        if geom.geom_type == "LineString":
            coords = list(geom.coords)
            desenhar_segmento(coords, cor_principal)
        elif geom.geom_type == "MultiLineString":
            # Para MultiLineString, desenha cada parte
            for part in geom.geoms:
                coords = list(part.coords)
                desenhar_segmento(coords, cor_principal)
        else:
            print(f"   ⚠️ Geometria ignorada: {geom.geom_type}")
            continue
```

### Realçamos os segmentos das vias de interesse


```python
def desenhar_segmento(coords, cor):
    """Desenha um segmento individual sem marcadores extras"""
    
    # Linha do segmento
    folium.PolyLine(
        locations=[(lat, lon) for lon, lat in coords],
        color=cor,
        weight=4,
        opacity=0.6,
    ).add_to(mapa)

# 9. Adiciona as vias ao mapa
print("\n📊 Processando Rua Edson Arantes...")
adicionar_via_completa(
    segmentos_edson_orig, 
    segmentos_edson_utm,
    "Rua Edson Arantes do Nascimento", 
    "blue"
)

print("\n📊 Processando Avenida Rei Pelé...")
adicionar_via_completa(
    segmentos_rei_orig,
    segmentos_rei_utm,
    "Avenida Rei Pelé", 
    "red"
)

```

    
    📊 Processando Rua Edson Arantes...
        Desenhando 5 segmentos individuais...
    
    📊 Processando Avenida Rei Pelé...
       📍 Via principal identificada (mais longa)
        Desenhando 74 segmentos individuais...


### Adicionamos um ponto de interesse especial


```python
casa_pelé_coords = (-21.694709, -45.255379)

folium.Marker(
    location=[casa_pelé_coords[0], casa_pelé_coords[1]],
    popup="""
    <b>🏠 Casa Pelé</b><br>
    <i>Local onde nasceu o Rei do futebol</i><br>
    """,
    tooltip="🏠 Casa Pelé",
    icon=folium.Icon(
        color="gold",
        icon="home",
        prefix="fa"
    )
).add_to(mapa)
```




    <folium.map.Marker at 0x782c4d110800>



### Adicionamos legendas


```python
legend_html = """
<div style="position: fixed; bottom: 20px; right: 20px; z-index: 1000; 
            background-color: white; padding: 15px; border-radius: 8px; 
            border: 2px solid #444; font-family: Arial; font-size: 13px;
            max-width: 320px; box-shadow: 3px 3px 10px rgba(0,0,0,0.3);">
    <b>🗺️ Legenda</b><br><br>
    <span style="color: blue; font-weight: bold;">■</span> Rua Edson Arantes<br>
    <span style="color: red; font-weight: bold;">■</span> Avenida Rei Pelé<br>
    <span style="color: gold; font-weight: bold;">🏠</span> Casa Pelé<br><br>
       
    <b>Marcadores:</b><br>
    <span style="color: gray;">●</span> A extensão das vias é definida pelas cores<br><br>
    
    <span style="font-size: 11px; color: #666;">
    ✅ Total: 74 segmentos (Avenida Rei Pelé)<br>
    ✅ Total: 5 segmentos (Rua Edson Arantes)<br>
    ⚠️ Pistas duplas aparecem separadas
    </span>
</div>
"""
mapa.get_root().html.add_child(folium.Element(legend_html))
```




    <branca.element.Element at 0x782c4d2f9ca0>



### Adicionamos controle de camadas e plugins


```python
folium.LayerControl().add_to(mapa)
MeasureControl().add_to(mapa)
Fullscreen().add_to(mapa)
```




    <folium.plugins.fullscreen.Fullscreen at 0x782c4d2fb7d0>



### Visualizamos o mapa


```python
display(mapa)
```


<div style="width:100%;"><div style="position:relative;width:100%;height:0;padding-bottom:60%;"><span style="color:#565656">Make this Notebook Trusted to load map: File -> Trust Notebook</span><iframe srcdoc="&lt;!DOCTYPE html&gt;
&lt;html&gt;
&lt;head&gt;

    &lt;meta http-equiv=&quot;content-type&quot; content=&quot;text/html; charset=UTF-8&quot; /&gt;
    &lt;script src=&quot;https://cdn.jsdelivr.net/npm/leaflet@1.9.3/dist/leaflet.js&quot;&gt;&lt;/script&gt;
    &lt;script src=&quot;https://code.jquery.com/jquery-3.7.1.min.js&quot;&gt;&lt;/script&gt;
    &lt;script src=&quot;https://cdn.jsdelivr.net/npm/bootstrap@5.2.2/dist/js/bootstrap.bundle.min.js&quot;&gt;&lt;/script&gt;
    &lt;script src=&quot;https://cdnjs.cloudflare.com/ajax/libs/Leaflet.awesome-markers/2.0.2/leaflet.awesome-markers.js&quot;&gt;&lt;/script&gt;
    &lt;link rel=&quot;stylesheet&quot; href=&quot;https://cdn.jsdelivr.net/npm/leaflet@1.9.3/dist/leaflet.css&quot;/&gt;
    &lt;link rel=&quot;stylesheet&quot; href=&quot;https://cdn.jsdelivr.net/npm/bootstrap@5.2.2/dist/css/bootstrap.min.css&quot;/&gt;
    &lt;link rel=&quot;stylesheet&quot; href=&quot;https://netdna.bootstrapcdn.com/bootstrap/3.0.0/css/bootstrap-glyphicons.css&quot;/&gt;
    &lt;link rel=&quot;stylesheet&quot; href=&quot;https://cdn.jsdelivr.net/npm/@fortawesome/fontawesome-free@6.2.0/css/all.min.css&quot;/&gt;
    &lt;link rel=&quot;stylesheet&quot; href=&quot;https://cdnjs.cloudflare.com/ajax/libs/Leaflet.awesome-markers/2.0.2/leaflet.awesome-markers.css&quot;/&gt;
    &lt;link rel=&quot;stylesheet&quot; href=&quot;https://cdn.jsdelivr.net/gh/python-visualization/folium/folium/templates/leaflet.awesome.rotate.min.css&quot;/&gt;

            &lt;meta name=&quot;viewport&quot; content=&quot;width=device-width,
                initial-scale=1.0, maximum-scale=1.0, user-scalable=no&quot; /&gt;
            &lt;style&gt;
                #map_d0ec70f8ba0eb50a7c405f84287389b9 {
                    position: relative;
                    width: 100.0%;
                    height: 100.0%;
                    left: 0.0%;
                    top: 0.0%;
                }
                .leaflet-container { font-size: 1rem; }
            &lt;/style&gt;

            &lt;style&gt;html, body {
                width: 100%;
                height: 100%;
                margin: 0;
                padding: 0;
            }
            &lt;/style&gt;

            &lt;style&gt;#map {
                position:absolute;
                top:0;
                bottom:0;
                right:0;
                left:0;
                }
            &lt;/style&gt;

            &lt;script&gt;
                L_NO_TOUCH = false;
                L_DISABLE_3D = false;
            &lt;/script&gt;


    &lt;script src=&quot;https://cdn.jsdelivr.net/gh/ljagis/leaflet-measure@2.1.7/dist/leaflet-measure.min.js&quot;&gt;&lt;/script&gt;
    &lt;link rel=&quot;stylesheet&quot; href=&quot;https://cdn.jsdelivr.net/gh/ljagis/leaflet-measure@2.1.7/dist/leaflet-measure.min.css&quot;/&gt;
    &lt;script src=&quot;https://cdn.jsdelivr.net/npm/leaflet.fullscreen@3.0.0/Control.FullScreen.min.js&quot;&gt;&lt;/script&gt;
    &lt;link rel=&quot;stylesheet&quot; href=&quot;https://cdn.jsdelivr.net/npm/leaflet.fullscreen@3.0.0/Control.FullScreen.css&quot;/&gt;
&lt;/head&gt;
&lt;body&gt;


&lt;div style=&quot;position: fixed; bottom: 20px; right: 20px; z-index: 1000; 
            background-color: white; padding: 15px; border-radius: 8px; 
            border: 2px solid #444; font-family: Arial; font-size: 13px;
            max-width: 320px; box-shadow: 3px 3px 10px rgba(0,0,0,0.3);&quot;&gt;
    &lt;b&gt;🗺️ Legenda&lt;/b&gt;&lt;br&gt;&lt;br&gt;
    &lt;span style=&quot;color: blue; font-weight: bold;&quot;&gt;■&lt;/span&gt; Rua Edson Arantes&lt;br&gt;
    &lt;span style=&quot;color: red; font-weight: bold;&quot;&gt;■&lt;/span&gt; Avenida Rei Pelé&lt;br&gt;
    &lt;span style=&quot;color: gold; font-weight: bold;&quot;&gt;🏠&lt;/span&gt; Casa Pelé&lt;br&gt;&lt;br&gt;

    &lt;b&gt;Marcadores:&lt;/b&gt;&lt;br&gt;
    &lt;span style=&quot;color: gray;&quot;&gt;●&lt;/span&gt; A extensão das vias é definida pelas cores&lt;br&gt;&lt;br&gt;

    &lt;span style=&quot;font-size: 11px; color: #666;&quot;&gt;
    ✅ Total: 74 segmentos (Avenida Rei Pelé)&lt;br&gt;
    ✅ Total: 5 segmentos (Rua Edson Arantes)&lt;br&gt;
    ⚠️ Pistas duplas aparecem separadas
    &lt;/span&gt;
&lt;/div&gt;

            &lt;div class=&quot;folium-map&quot; id=&quot;map_d0ec70f8ba0eb50a7c405f84287389b9&quot; &gt;&lt;/div&gt;

&lt;/body&gt;
&lt;script&gt;


            var map_d0ec70f8ba0eb50a7c405f84287389b9 = L.map(
                &quot;map_d0ec70f8ba0eb50a7c405f84287389b9&quot;,
                {
                    center: [-21.697, -45.253],
                    crs: L.CRS.EPSG3857,
                    ...{
  &quot;zoom&quot;: 15,
  &quot;zoomControl&quot;: true,
  &quot;preferCanvas&quot;: false,
}

                }
            );





            var tile_layer_06eb9346948683803720e19451e14492 = L.tileLayer(
                &quot;https://tile.openstreetmap.org/{z}/{x}/{y}.png&quot;,
                {
  &quot;minZoom&quot;: 0,
  &quot;maxZoom&quot;: 19,
  &quot;maxNativeZoom&quot;: 19,
  &quot;noWrap&quot;: false,
  &quot;attribution&quot;: &quot;\u0026copy; \u003ca href=\&quot;https://www.openstreetmap.org/copyright\&quot;\u003eOpenStreetMap\u003c/a\u003e contributors&quot;,
  &quot;subdomains&quot;: &quot;abc&quot;,
  &quot;detectRetina&quot;: false,
  &quot;tms&quot;: false,
  &quot;opacity&quot;: 1,
}

            );


            tile_layer_06eb9346948683803720e19451e14492.addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_e1edf7febd6edea59add64e7030a42b2 = L.polyline(
                [[-21.6942219, -45.2537301], [-21.6944002, -45.2539243], [-21.6945496, -45.2541401], [-21.6946099, -45.2542755], [-21.6946389, -45.2543986], [-21.6946566, -45.2544875], [-21.6947585, -45.2550298], [-21.694779, -45.2551369], [-21.6949183, -45.2558657], [-21.6949464, -45.2560142], [-21.6949945, -45.2562675], [-21.6950883, -45.2567613], [-21.6950956, -45.2568], [-21.6951538, -45.2571065], [-21.6951756, -45.2572214], [-21.6951843, -45.257264], [-21.6953041, -45.2578572], [-21.6955156, -45.2588666], [-21.6955146, -45.2589807], [-21.6955116, -45.2589964], [-21.6954927, -45.2590473]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;blue&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;blue&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.8, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 10}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


        var popup_de8bc2181ff23d893fb02abe9f7be0db = L.popup({
  &quot;maxWidth&quot;: &quot;100%&quot;,
});



                var html_5c5c74e7d04fffd4ce519e2e280dfe65 = $(`&lt;div id=&quot;html_5c5c74e7d04fffd4ce519e2e280dfe65&quot; style=&quot;width: 100.0%; height: 100.0%;&quot;&gt;Rua Edson Arantes do Nascimento (VIA PRINCIPAL)&lt;/div&gt;`)[0];
                popup_de8bc2181ff23d893fb02abe9f7be0db.setContent(html_5c5c74e7d04fffd4ce519e2e280dfe65);



        poly_line_e1edf7febd6edea59add64e7030a42b2.bindPopup(popup_de8bc2181ff23d893fb02abe9f7be0db)
        ;




            poly_line_e1edf7febd6edea59add64e7030a42b2.bindTooltip(
                `&lt;div&gt;
                     Rua Edson Arantes do Nascimento
                 &lt;/div&gt;`,
                {
  &quot;sticky&quot;: true,
}
            );


            var poly_line_ccd97b88b34f9d81b3bbb4b20d3f9c87 = L.polyline(
                [[-21.6947585, -45.2550298], [-21.694779, -45.2551369], [-21.6949183, -45.2558657], [-21.6949464, -45.2560142], [-21.6949945, -45.2562675], [-21.6950883, -45.2567613], [-21.6950956, -45.2568], [-21.6951538, -45.2571065], [-21.6951756, -45.2572214], [-21.6951843, -45.257264]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;blue&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;blue&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_e684b032fabd21cecb59893493d50632 = L.polyline(
                [[-21.6951843, -45.257264], [-21.6953041, -45.2578572], [-21.6955156, -45.2588666]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;blue&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;blue&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_575a89cecee5acf4c24a686fafbfdb43 = L.polyline(
                [[-21.6942219, -45.2537301], [-21.6944002, -45.2539243], [-21.6945496, -45.2541401], [-21.6946099, -45.2542755], [-21.6946389, -45.2543986], [-21.6946566, -45.2544875]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;blue&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;blue&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_7d87ebed5cf4246784977d74b6af948f = L.polyline(
                [[-21.6955156, -45.2588666], [-21.6955146, -45.2589807], [-21.6955116, -45.2589964], [-21.6954927, -45.2590473]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;blue&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;blue&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_0542a33d92acb2a2ea581b78c670c627 = L.polyline(
                [[-21.6946566, -45.2544875], [-21.6947585, -45.2550298]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;blue&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;blue&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_a40b9867c1b58542561877bc6b3b895c = L.polyline(
                [[-21.6645225, -45.2800996], [-21.6643793, -45.2804923], [-21.663607, -45.2819065], [-21.6631891, -45.2825908], [-21.6628819, -45.283134], [-21.6627286, -45.2834029], [-21.6625615, -45.2836999], [-21.6622655, -45.284229], [-21.6620312, -45.2846468], [-21.661838, -45.2849914], [-21.6615673, -45.2854854], [-21.6615079, -45.2855895], [-21.6614544, -45.2856834], [-21.6613114, -45.2859322], [-21.6611936, -45.2861528], [-21.6609967, -45.2866128], [-21.6608527, -45.2870125], [-21.6607293, -45.2873819], [-21.6606552, -45.2876575], [-21.660581, -45.2879968], [-21.6605255, -45.2882805], [-21.6604308, -45.2887452], [-21.6603754, -45.2890395], [-21.6602191, -45.2898478], [-21.6602012, -45.2899443], [-21.6599541, -45.2912175], [-21.659797, -45.2919672], [-21.6592931, -45.2946192], [-21.6592886, -45.2946424], [-21.6588663, -45.2968085], [-21.658818, -45.297056]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.8, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 10}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


        var popup_d540e6424340979c26606f27498621dd = L.popup({
  &quot;maxWidth&quot;: &quot;100%&quot;,
});



                var html_522e91c9048570376b0d293396e19ff2 = $(`&lt;div id=&quot;html_522e91c9048570376b0d293396e19ff2&quot; style=&quot;width: 100.0%; height: 100.0%;&quot;&gt;Avenida Rei Pelé (VIA PRINCIPAL)&lt;/div&gt;`)[0];
                popup_d540e6424340979c26606f27498621dd.setContent(html_522e91c9048570376b0d293396e19ff2);



        poly_line_a40b9867c1b58542561877bc6b3b895c.bindPopup(popup_d540e6424340979c26606f27498621dd)
        ;




            poly_line_a40b9867c1b58542561877bc6b3b895c.bindTooltip(
                `&lt;div&gt;
                     Avenida Rei Pelé
                 &lt;/div&gt;`,
                {
  &quot;sticky&quot;: true,
}
            );


            var poly_line_e9e0305dd0491ec3eac7848d20bf6daf = L.polyline(
                [[-21.66568, -45.2768442], [-21.6657591, -45.2764533], [-21.6658264, -45.2762139], [-21.665908, -45.2759893], [-21.6660682, -45.2756359], [-21.6662458, -45.2753033], [-21.6663384, -45.2751677], [-21.6665026, -45.2749687], [-21.6666901, -45.2747635], [-21.6668477, -45.274606], [-21.6669326, -45.2745409], [-21.6671201, -45.2743953], [-21.6673164, -45.2742679], [-21.6676377, -45.2740731]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_2f7396e96f458ff3ff2b83aeb6cecd16 = L.polyline(
                [[-21.684644, -45.2635715], [-21.6846213, -45.2635982], [-21.6842924, -45.263943], [-21.6839003, -45.2642987], [-21.6838203, -45.2643671], [-21.6837413, -45.2644264]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_e09d2f7002bef5cc8c043505df09f1e9 = L.polyline(
                [[-21.6814943, -45.2661765], [-21.6817672, -45.2660022], [-21.6821794, -45.2657218], [-21.6823317, -45.2656149], [-21.6828684, -45.2652712], [-21.683002, -45.2651919], [-21.6835686, -45.2648045]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_ad25f161e2ae715c7a02210ff8306a7b = L.polyline(
                [[-21.6591488, -45.2970252], [-21.6590929, -45.2969631], [-21.6590179, -45.2969239], [-21.658943, -45.2969239], [-21.6588884, -45.2969704], [-21.658818, -45.297056]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_0a7efbc11e58c6b3ba89a47c5df848a7 = L.polyline(
                [[-21.66632, -45.2750179], [-21.6661071, -45.2753436], [-21.6660601, -45.2754073], [-21.6660252, -45.2754615], [-21.6658987, -45.2757385], [-21.6657903, -45.2759926], [-21.6657005, -45.2762736], [-21.6656338, -45.2765552], [-21.665513, -45.2771688], [-21.6653092, -45.2782101], [-21.6652288, -45.2785293], [-21.6651758, -45.278711], [-21.6650292, -45.2791554], [-21.6649549, -45.2793407], [-21.664886, -45.2794976], [-21.6647171, -45.2797752], [-21.6646841, -45.2798215], [-21.6645955, -45.2799037]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_12e0b58e8abf64028687b174883db4f3 = L.polyline(
                [[-21.6733757, -45.2715668], [-21.6731988, -45.2717153], [-21.673115, -45.2717768], [-21.6729098, -45.2719248], [-21.6727552, -45.2719737], [-21.6727226, -45.2719813], [-21.6726755, -45.2719872], [-21.6726257, -45.272022]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_625b87c9bc5e6ceec68da500c2cde7fc = L.polyline(
                [[-21.6587336, -45.2975318], [-21.6587296, -45.2977054], [-21.6587603, -45.2977672], [-21.6588145, -45.2978373], [-21.6588899, -45.297877]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_60e55b046c065d2551bbe29672939e44 = L.polyline(
                [[-21.6617648, -45.285336], [-21.6619084, -45.2850524], [-21.6620985, -45.284697], [-21.6623546, -45.2842578], [-21.6625042, -45.2839868], [-21.6626469, -45.2837428], [-21.6628093, -45.2834336], [-21.6629563, -45.2831829], [-21.6636222, -45.2820762]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_34a44d88c933a37860e164c382787e27 = L.polyline(
                [[-21.6614544, -45.2856834], [-21.6613114, -45.2859322], [-21.6611936, -45.2861528], [-21.6609967, -45.2866128], [-21.6608527, -45.2870125], [-21.6607293, -45.2873819], [-21.6606552, -45.2876575], [-21.660581, -45.2879968], [-21.6605255, -45.2882805], [-21.6604308, -45.2887452]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_04364eb2b5e04dc40e410d1e8d4f52ff = L.polyline(
                [[-21.6725085, -45.2721097], [-21.6725053, -45.2721551], [-21.6725183, -45.2721985], [-21.6725454, -45.2722336], [-21.6725827, -45.2722551], [-21.6726247, -45.2722601], [-21.6726411, -45.2722573], [-21.672671, -45.2722446], [-21.6726963, -45.2722231], [-21.6727147, -45.2721948], [-21.6727182, -45.2721866], [-21.6727264, -45.2721481], [-21.6727225, -45.2721088], [-21.6727069, -45.2720731], [-21.6726814, -45.2720448], [-21.6726257, -45.272022], [-21.6725786, -45.2720284], [-21.6725383, -45.2720553], [-21.6725121, -45.2720979], [-21.6725102, -45.2721035], [-21.6725085, -45.2721097]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_5e117abecf245a162b23d344b97fbf92 = L.polyline(
                [[-21.6602191, -45.2898478], [-21.6602012, -45.2899443], [-21.6599541, -45.2912175], [-21.659797, -45.2919672], [-21.6592931, -45.2946192]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_5bedea06a3a98888c962840a9ac4c705 = L.polyline(
                [[-21.6588899, -45.297877], [-21.6589498, -45.2977644], [-21.6590476, -45.297585], [-21.6591052, -45.2974584], [-21.6591545, -45.297341]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_d7a9faec4dd13db1b723f3ec999aeeb2 = L.polyline(
                [[-21.6604308, -45.2887452], [-21.6603754, -45.2890395]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_cb9502d340f5c781c1903f60f7d44ee2 = L.polyline(
                [[-21.6604701, -45.2890837], [-21.660531, -45.2887789]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_59b3b588ce7b6e758e702cd30e6869b8 = L.polyline(
                [[-21.6733195, -45.2717763], [-21.6735096, -45.2716032], [-21.6738041, -45.2713277], [-21.6741183, -45.2710009]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_9083d4d5dc9f67d8547386b5f2ad515a = L.polyline(
                [[-21.6773465, -45.2688028], [-21.6773879, -45.2687801], [-21.6774183, -45.2687422], [-21.6774295, -45.2687135]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_40d59d2504e8e1a1907701f5d9ec90c2 = L.polyline(
                [[-21.684864, -45.2634756], [-21.6849159, -45.2634075], [-21.6852617, -45.2629811], [-21.6856701, -45.2624595], [-21.6861063, -45.2619649], [-21.6863602, -45.261693], [-21.6867573, -45.2613313], [-21.6868918, -45.2612089]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_9ebbbd885ce030b3b28c08f87bf6139e = L.polyline(
                [[-21.6587336, -45.2975318], [-21.6585656, -45.2985423]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_027eb515adbc56530bf119c9b014a542 = L.polyline(
                [[-21.658818, -45.297056], [-21.6587336, -45.2975318]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_f84e7e79a5d4f918530403adb77023b0 = L.polyline(
                [[-21.6591488, -45.2970252], [-21.6590994, -45.296536], [-21.6591163, -45.2962552], [-21.65913, -45.296121], [-21.6591437, -45.2959861]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_190f40a282f4b801f64fe70cfffcd69f = L.polyline(
                [[-21.6581757, -45.2993788], [-21.658296, -45.2992055], [-21.6583962, -45.299043], [-21.6584885, -45.2988114], [-21.6585656, -45.2985423]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_0ecdc100c9f256ee07247c8dfc9df396 = L.polyline(
                [[-21.6883225, -45.2603862], [-21.6888137, -45.2601949], [-21.6890152, -45.2600645], [-21.6890423, -45.2600403]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_d817f50adc89c83b1abd270b575006f3 = L.polyline(
                [[-21.6884691, -45.2602076], [-21.6882322, -45.2603076], [-21.688017, -45.2603975]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_2cec6be63762239c57cfd10b362aeb22 = L.polyline(
                [[-21.6877918, -45.2604632], [-21.6876183, -45.2605657], [-21.6875421, -45.2606067], [-21.6873427, -45.2607207], [-21.6871371, -45.2608734], [-21.6868345, -45.261144]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_5e3701b5d2dd7dc56a98840ddb2cb0b5 = L.polyline(
                [[-21.6571053, -45.3002459], [-21.6578271, -45.2997168], [-21.6581757, -45.2993788]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_6a763bf5769a99fde28b154592d9dfad = L.polyline(
                [[-21.6585656, -45.2985423], [-21.658711, -45.2981862], [-21.6588899, -45.297877]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_d5cf4e1dc2f7fcf577c17642a9ea75ff = L.polyline(
                [[-21.6603754, -45.2890395], [-21.6602191, -45.2898478]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_99dd95ee1b923ff10da23432ebe62e8a = L.polyline(
                [[-21.663607, -45.2819065], [-21.6631891, -45.2825908], [-21.6628819, -45.283134], [-21.6627286, -45.2834029], [-21.6625615, -45.2836999], [-21.6622655, -45.284229], [-21.6620312, -45.2846468], [-21.661838, -45.2849914], [-21.6615673, -45.2854854], [-21.6615079, -45.2855895], [-21.6614544, -45.2856834]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_fc61e8a334413475e6c2af71ec7c21ca = L.polyline(
                [[-21.6645225, -45.2800996], [-21.6643793, -45.2804923], [-21.663607, -45.2819065]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_e97661c7044ee9be21a0472430d12042 = L.polyline(
                [[-21.6725085, -45.2721097], [-21.6723535, -45.2722397], [-21.6722288, -45.2723234], [-21.6719342, -45.2724253], [-21.6704224, -45.272892], [-21.6701212, -45.272974], [-21.6691221, -45.273257]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_49ac04f45485d7fa807864c336e7fd9d = L.polyline(
                [[-21.6753056, -45.2699074], [-21.6747153, -45.2702904], [-21.6743813, -45.2705284], [-21.6737958, -45.2711557], [-21.6733757, -45.2715668]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_cff9e33143cc5ce70ebbe282c7a6b5ea = L.polyline(
                [[-21.6772041, -45.2686255], [-21.6771283, -45.2687], [-21.677056, -45.2687603], [-21.6768138, -45.2689337], [-21.6767589, -45.2689738], [-21.6756652, -45.2696858], [-21.6754349, -45.2698264], [-21.6753056, -45.2699074]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_cfb32aff0d534e1d7c8a3c0c6aebf96b = L.polyline(
                [[-21.6784703, -45.2678769], [-21.6783253, -45.2679695], [-21.6781761, -45.2680648], [-21.6777867, -45.2683236], [-21.6775592, -45.2684489], [-21.6774013, -45.2685267], [-21.6773375, -45.2685515]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_6162be48f90d6fc01a3a547b18795c5e = L.polyline(
                [[-21.6868345, -45.261144], [-21.6867084, -45.2612478], [-21.6860649, -45.2618641], [-21.6855559, -45.2624359], [-21.6850115, -45.2631351], [-21.684793, -45.2633956], [-21.684644, -45.2635715]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_60bdff150239dec63e73e503a9cf1ad4 = L.polyline(
                [[-21.688017, -45.2603975], [-21.6877918, -45.2604632]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_cbd4824c673cde5a5e785ea7022c277c = L.polyline(
                [[-21.6880444, -45.2604913], [-21.6882624, -45.2604074], [-21.6883225, -45.2603862]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_8aba5c8f6fc5e1d147f15ffae3367ef9 = L.polyline(
                [[-21.6835583, -45.2647019], [-21.6837019, -45.2645971], [-21.6837923, -45.264522], [-21.6839578, -45.2643812], [-21.6840989, -45.2642571], [-21.6843573, -45.2640224], [-21.6846545, -45.2637174]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_1f5b155a190031a823f8df077c4a6944 = L.polyline(
                [[-21.6783349, -45.2680995], [-21.6783792, -45.2680706], [-21.6786965, -45.2678686], [-21.6787528, -45.2678318], [-21.6797995, -45.2671477], [-21.6800214, -45.2669987], [-21.6801618, -45.2669045], [-21.6805575, -45.266674], [-21.6811453, -45.2662679]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_3d98787a27f1a17958930ed0c73090ad = L.polyline(
                [[-21.6774295, -45.2687135], [-21.6776774, -45.2685338], [-21.677798, -45.2684469], [-21.6778946, -45.2683844], [-21.6783349, -45.2680995]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_43902a5b000d2dd5cf6eca5c56f40e44 = L.polyline(
                [[-21.6727182, -45.2721866], [-21.6729071, -45.2720613], [-21.6730689, -45.271955], [-21.6731491, -45.271898], [-21.6732587, -45.2718222], [-21.6733195, -45.2717763]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_32f1054a941dc6b7b26cd0ba30a8ab57 = L.polyline(
                [[-21.6676377, -45.2740731], [-21.6679321, -45.2738991], [-21.6681801, -45.2737731], [-21.6685341, -45.2736081], [-21.6687503, -45.2735116], [-21.6690108, -45.2734143], [-21.6692813, -45.2733245]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_21b0372ed88e8234506b4b248af55ea8 = L.polyline(
                [[-21.6647775, -45.28004], [-21.6648705, -45.2798952], [-21.6649216, -45.2797464], [-21.665116, -45.2791831], [-21.6651787, -45.2789953]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_afb1ceba7f987c464b5f0a7e7a4b5570 = L.polyline(
                [[-21.6641525, -45.2811184], [-21.664256, -45.2809405], [-21.6643507, -45.280786], [-21.6644828, -45.2805745], [-21.6646478, -45.2803086], [-21.6646664, -45.2802664], [-21.6646873, -45.2802187], [-21.664708, -45.2801645]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_df087ff3e02614384b5f42a84aaf676f = L.polyline(
                [[-21.6606636, -45.2880421], [-21.6607437, -45.2876683], [-21.6608191, -45.28739], [-21.6609394, -45.2870426], [-21.661074, -45.2866772], [-21.6612765, -45.2862132], [-21.6613837, -45.2859946], [-21.661593, -45.2856301]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_650fb1399268302843a6480817180c5f = L.polyline(
                [[-21.6599692, -45.291691], [-21.6599989, -45.29154], [-21.6602863, -45.2900565]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_2296ec19f7668290764c4c629e0800cd = L.polyline(
                [[-21.6591437, -45.2959861], [-21.6592216, -45.2955585], [-21.6592804, -45.295236], [-21.6599692, -45.291691]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_8ba57c9e374afc25e62dabe1806354e0 = L.polyline(
                [[-21.6810267, -45.2662401], [-21.680907, -45.2663196], [-21.6797499, -45.2670591], [-21.6796127, -45.2671467], [-21.6794001, -45.2672826], [-21.6788605, -45.2676286], [-21.6788149, -45.2676566], [-21.6786937, -45.267734], [-21.6784703, -45.2678769]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_55875e96725c465e51b9ec6410b67d5a = L.polyline(
                [[-21.6691221, -45.273257], [-21.6689603, -45.2733258], [-21.6687242, -45.2734177], [-21.6684948, -45.2735216], [-21.6682531, -45.2736323], [-21.6680187, -45.2737469], [-21.6678062, -45.2738602], [-21.6674274, -45.2740849], [-21.6671637, -45.2742498], [-21.6669855, -45.2743732], [-21.6667205, -45.2745841], [-21.6665294, -45.2747802], [-21.66632, -45.2750179]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_3475e8da956a14a77f82a375dc7d2771 = L.polyline(
                [[-21.6592931, -45.2946192], [-21.6592886, -45.2946424], [-21.6588663, -45.2968085], [-21.658818, -45.297056]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_ec3b6c20efe3075827787662a8b11f1a = L.polyline(
                [[-21.6868918, -45.2612089], [-21.6872414, -45.2609057], [-21.6873969, -45.2607952], [-21.6874786, -45.2607464], [-21.6875815, -45.2606858], [-21.68765, -45.2606543], [-21.6878193, -45.2605873], [-21.6880444, -45.2604913]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_bce49554b292b0d072973091a17dcca5 = L.polyline(
                [[-21.6837413, -45.2644264], [-21.6834708, -45.2646379], [-21.6832077, -45.2648147], [-21.6825049, -45.2652833], [-21.6813925, -45.2659955]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_8c7c042f7cd58ada0512ad2b4739dbc8 = L.polyline(
                [[-21.6744337, -45.2706633], [-21.6746092, -45.2705213], [-21.6747789, -45.2704017], [-21.675245, -45.2700987], [-21.6757372, -45.2697707], [-21.6762355, -45.269445]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_6b7546c0c83fe0ad59f539eb1b685777 = L.polyline(
                [[-21.6591545, -45.297341], [-21.6591525, -45.2971902], [-21.6591488, -45.2970252]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_3d26a1c3ba3af192695e6c9b5a0d1936 = L.polyline(
                [[-21.6814378, -45.2660768], [-21.6816677, -45.2659265], [-21.6821676, -45.2655997], [-21.6829472, -45.2650901], [-21.6832137, -45.2649159], [-21.6835583, -45.2647019]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_ad7051c0195f9fe60b92a735115fb367 = L.polyline(
                [[-21.6846545, -45.2637174], [-21.6846881, -45.2636823], [-21.684864, -45.2634756]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_b1bb6f4ff977af0b5d3de9a38a9ccce0 = L.polyline(
                [[-21.6811453, -45.2662679], [-21.6811871, -45.2662406], [-21.6813859, -45.2661106], [-21.6814378, -45.2660768]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_bbb81d2a5c3c791ed25a518279b9f00f = L.polyline(
                [[-21.6813925, -45.2659955], [-21.6811503, -45.2661579], [-21.6810267, -45.2662401]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_f77de77fafc5747c19b517f3ed92bfa0 = L.polyline(
                [[-21.6636222, -45.2820762], [-21.6641525, -45.2811184]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_c1c45f4d1e34c8d42bd37c911f19540f = L.polyline(
                [[-21.660531, -45.2887789], [-21.6606636, -45.2880421]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_cc9ed5825475ea696b4e718da718137a = L.polyline(
                [[-21.661593, -45.2856301], [-21.6616428, -45.2855438], [-21.6617648, -45.285336]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_3f81f2acd6e9100a820871fed392109a = L.polyline(
                [[-21.6692813, -45.2733245], [-21.6699119, -45.2731359], [-21.6701388, -45.2730653], [-21.6709067, -45.2728554]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_0bbc072a6968995966370e5d4d576543 = L.polyline(
                [[-21.6713476, -45.2727178], [-21.6714864, -45.2726756]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_13740f6efdea0119f87b029b0a9eb7e4 = L.polyline(
                [[-21.6651787, -45.2789953], [-21.66523, -45.2788431], [-21.6652668, -45.2787348], [-21.6652894, -45.2786693], [-21.6653977, -45.2782309], [-21.6655379, -45.2775617], [-21.66568, -45.2768442]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_cd8c1a45684e7954e497980bc65d34f2 = L.polyline(
                [[-21.6741183, -45.2710009], [-21.6744337, -45.2706633]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_0a170f7aa8941d2f1f401d5f13a16ec9 = L.polyline(
                [[-21.6721436, -45.2724706], [-21.6723067, -45.2724177], [-21.672518, -45.2723511], [-21.6726247, -45.2722601]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_db38b505ec540fa5160bd2393a990a31 = L.polyline(
                [[-21.6709067, -45.2728554], [-21.6713476, -45.2727178]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_30bb6007f4a4b0fa440c97d352592b55 = L.polyline(
                [[-21.6714864, -45.2726756], [-21.6715972, -45.2726411], [-21.6716917, -45.272617], [-21.6719348, -45.2725305], [-21.6721436, -45.2724706]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_4266b9c10a18ebfab835dc9a9690b8ce = L.polyline(
                [[-21.6602863, -45.2900565], [-21.6604701, -45.2890837]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_7d248bb0347e5cec5bde2172638074ab = L.polyline(
                [[-21.6772041, -45.2686255], [-21.6771941, -45.2686664], [-21.677197, -45.2687086], [-21.6772123, -45.2687475], [-21.6772385, -45.2687791], [-21.6772727, -45.2687998]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_8862ed3affe2375533dc3ec67ace3280 = L.polyline(
                [[-21.6774295, -45.2687135], [-21.6774338, -45.2686713], [-21.6774253, -45.2686299], [-21.6774047, -45.2685937], [-21.6773744, -45.2685666], [-21.6773375, -45.2685515]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_26dc9b04a3f4072eb5ccf50927e338d2 = L.polyline(
                [[-21.6773375, -45.2685515], [-21.6773053, -45.2685493], [-21.6772738, -45.2685564]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_b6acec5dc18e178e3c9dd096edd73081 = L.polyline(
                [[-21.6772738, -45.2685564], [-21.6772119, -45.2686098], [-21.6772041, -45.2686255]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_163d1d31abeaa7ea546cb380ce191cef = L.polyline(
                [[-21.6772727, -45.2687998], [-21.6772969, -45.2688063], [-21.6773219, -45.2688073], [-21.6773465, -45.2688028]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var poly_line_3c620cfa3451c71254b66534cd541462 = L.polyline(
                [[-21.6762355, -45.269445], [-21.6763778, -45.2693574], [-21.6768662, -45.2690357], [-21.6771881, -45.2688435], [-21.6772727, -45.2687998]],
                {&quot;bubblingMouseEvents&quot;: true, &quot;color&quot;: &quot;red&quot;, &quot;dashArray&quot;: null, &quot;dashOffset&quot;: null, &quot;fill&quot;: false, &quot;fillColor&quot;: &quot;red&quot;, &quot;fillOpacity&quot;: 0.2, &quot;fillRule&quot;: &quot;evenodd&quot;, &quot;lineCap&quot;: &quot;round&quot;, &quot;lineJoin&quot;: &quot;round&quot;, &quot;noClip&quot;: false, &quot;opacity&quot;: 0.6, &quot;smoothFactor&quot;: 1.0, &quot;stroke&quot;: true, &quot;weight&quot;: 4}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var marker_c9fb6ebfce35f1e29ebdfee151c40f52 = L.marker(
                [-21.694709, -45.255379],
                {
}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);


            var icon_10c966a3fe083d18ecd338131acbae82 = L.AwesomeMarkers.icon(
                {
  &quot;markerColor&quot;: &quot;gold&quot;,
  &quot;iconColor&quot;: &quot;white&quot;,
  &quot;icon&quot;: &quot;home&quot;,
  &quot;prefix&quot;: &quot;fa&quot;,
  &quot;extraClasses&quot;: &quot;fa-rotate-0&quot;,
}
            );


        var popup_bfcee47e03c029534a63334d7418b390 = L.popup({
  &quot;maxWidth&quot;: &quot;100%&quot;,
});



                var html_912412bba89303c80f8800ef26cbd129 = $(`&lt;div id=&quot;html_912412bba89303c80f8800ef26cbd129&quot; style=&quot;width: 100.0%; height: 100.0%;&quot;&gt;     &lt;b&gt;🏠 Casa Pelé&lt;/b&gt;&lt;br&gt;     &lt;i&gt;Local onde nasceu o Rei do futebol&lt;/i&gt;&lt;br&gt;     &lt;/div&gt;`)[0];
                popup_bfcee47e03c029534a63334d7418b390.setContent(html_912412bba89303c80f8800ef26cbd129);



        marker_c9fb6ebfce35f1e29ebdfee151c40f52.bindPopup(popup_bfcee47e03c029534a63334d7418b390)
        ;




            marker_c9fb6ebfce35f1e29ebdfee151c40f52.bindTooltip(
                `&lt;div&gt;
                     🏠 Casa Pelé
                 &lt;/div&gt;`,
                {
  &quot;sticky&quot;: true,
}
            );


                marker_c9fb6ebfce35f1e29ebdfee151c40f52.setIcon(icon_10c966a3fe083d18ecd338131acbae82);


            var layer_control_541c57db592ed4396577957e8d527806_layers = {
                base_layers : {
                    &quot;openstreetmap&quot; : tile_layer_06eb9346948683803720e19451e14492,
                },
                overlays :  {
                },
            };
            let layer_control_541c57db592ed4396577957e8d527806 = L.control.layers(
                layer_control_541c57db592ed4396577957e8d527806_layers.base_layers,
                layer_control_541c57db592ed4396577957e8d527806_layers.overlays,
                {
  &quot;position&quot;: &quot;topright&quot;,
  &quot;collapsed&quot;: true,
  &quot;autoZIndex&quot;: true,
}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);



            var measure_control_740d1fb911d42ba327c88c5abe8d5e1d = new L.Control.Measure(
                {
  &quot;position&quot;: &quot;topright&quot;,
  &quot;primaryLengthUnit&quot;: &quot;meters&quot;,
  &quot;secondaryLengthUnit&quot;: &quot;miles&quot;,
  &quot;primaryAreaUnit&quot;: &quot;sqmeters&quot;,
  &quot;secondaryAreaUnit&quot;: &quot;acres&quot;,
});
            map_d0ec70f8ba0eb50a7c405f84287389b9.addControl(measure_control_740d1fb911d42ba327c88c5abe8d5e1d);

            // Workaround for using this plugin with Leaflet&gt;=1.8.0
            // https://github.com/ljagis/leaflet-measure/issues/171
            L.Control.Measure.include({
                _setCaptureMarkerIcon: function () {
                    // disable autopan
                    this._captureMarker.options.autoPanOnFocus = false;
                    // default function
                    this._captureMarker.setIcon(
                        L.divIcon({
                            iconSize: this._map.getSize().multiplyBy(2)
                        })
                    );
                },
            });



            L.control.fullscreen(
                {
  &quot;position&quot;: &quot;topleft&quot;,
  &quot;title&quot;: &quot;Full Screen&quot;,
  &quot;titleCancel&quot;: &quot;Exit Full Screen&quot;,
  &quot;forceSeparateButton&quot;: false,
}
            ).addTo(map_d0ec70f8ba0eb50a7c405f84287389b9);

&lt;/script&gt;
&lt;/html&gt;" style="position:absolute;width:100%;height:100%;left:0;top:0;border:none !important;" allowfullscreen webkitallowfullscreen mozallowfullscreen></iframe></div></div>


**Considerações finais:*

Demonstramos de forma didática como realizar o mapeamento e análise de vias urbanas utilizando **OSMnx**. A combinação de extração completa de dados, projeção métrica precisa e visualização interativa resulta em uma ferramenta poderosa para estudos urbanos, roteirização e análise de infraestrutura viária.

O estudo de caso apresentado, que serviu como exemplo prático, envolvendo a *Avenida Rei Pelé* e *Rua Edson Arantes do Nascimento*, em Três Corações/MG, permitiu a identifição de 74 e 5 segmentos respectivamente, com comprimentos totais de 11,8 km e 576 metros, respectivamente. A inclusão da **Casa Pelé** como ponto de interesse demonstra a flexibilidade da ferramenta para adicionar marcadores personalizados.


```python

```
