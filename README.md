# SeaCast_implementation_TFG
Este repositorio contiene todo el código utilizado para la realización del Trabajo de Fin de Grado del grado en Ciencia e Ingeniería de Datos por la Universidad de Las Palmas de Gran Canaria.

Este repositorio contiene la implementación del modelo **Hi-LAM** para la predicción de variables oceanográficas. 

SeaCast se basa en [Neural-LAM](https://github.com/mllam/neural-lam), un enfoque de predicción meteorológica en áreas limitadas impulsado por datos. Este repositorio contiene variaciones de malla similares a:

- El modelo basado en grafos propuesto por [Keisler (2022)](https://arxiv.org/abs/2202.07575).
- GraphCast, desarrollado por [Lam et al. (2023)](https://arxiv.org/abs/2212.12794).
- El modelo jerárquico propuesto por [Oskarsson et al. (2024)](https://arxiv.org/abs/2406.04759).
  
- La implementación original de SeaCast ha sido creada por [Deinal (2023)](https://github.com/deinal/seacast). Cuyos resultados se demuestran en [Holmberg et al. (2024)](https://arxiv.org/pdf/2410.11807).


## Ejecución del código

### Clonar el repositorio

```bash
git clone https://github.com/Selega6/SeaCast_implementation_TFG.git
```

### Crear entorno Conda
```bash
conda create -n seacast_env python=3.10.16
conda activate seacast_env
```

### Instalación de dependencias
```bash
pip install -r requirements.txt
```

### Descarga de los datos usados
Antes de descargar los datos, es necesario haber configurado las credenciales para la API de Copernicus en el archivo ~/.cdsapirc.
Se pueden seguir estas instrucciones apra este cometido: [Documenación API Copernicus](https://cds.climate.copernicus.eu/how-to-api)

Para descargar los datos utilizados para el Trabajo de Fin de Grado, descargaremos los datos comprendidos entre el 1 de enero de 2018 hasta el 31 de diciembre de 2023:

```bash
user=[CMEMS-usuario]
password=[CMEMS-contraseña]

python download_data.py --static -b data/atlantic/ -u $user -psw $password &&
python download_data.py -d reanalysis -s 2018-01-01 -e 2023-12-31 -u $user -psw $password &&
python download_data.py -d era5 -s 2018-01-01 -e 2023-12-31
```


### Preparación de las muestras

En esta parte, dividiremos el conjunto de datos de la siguiente manera:

- Entrenamiento: 2018-01-01 a 2021-12-31
- Validación: 2022-01-01 a 2022-12-31
- Test: 2023-01-01 a 2023-12-31

Para los datos oceanográficos:
#### Entrenamiento
```bash
python prepare_states.py -d data/atlantic/raw/reanalysis -o data/atlantic/samples/train -n 6 -p rea_data -s 2018-01-01 -e 2021-12-31
```
#### Validación
```bash
python prepare_states.py -d data/atlantic/raw/reanalysis -o data/atlantic/samples/val -n 6 -p rea_data -s 2022-01-01 -e 2022-12-31
```

#### Test
```bash
python prepare_states.py -d data/atlantic/raw/reanalysis -o data/atlantic/samples/test -n 17 -p rea_data -s 2023-01-01 -e 2023-12-31
```

Para los forzantes atmosféricos:
#### Entrenamiento
```bash
python prepare_states.py -d data/atlantic/raw/era5 -o data/atlantic/samples/train -n 6 -p forcing -s 2018-01-01 -e 2021-12-31
```

#### Validación
```bash
python prepare_states.py -d data/atlantic/raw/era5 -o data/atlantic/samples/val -n 6 -p forcing -s 2022-01-01 -e 2022-12-31
```

#### Test
```bash
python prepare_states.py -d data/atlantic/raw/era5 -o data/atlantic/samples/test -n 17 -p forcing -s 2023-01-01 -e 2023-12-31
```

### Generación de Características

#### Crear características de la rejilla
```bash
python create_grid_features.py --dataset atlantic
```

#### Generar pesos de los parámetros
```bash
python create_parameter_weights.py --dataset atlantic --batch_size 4 --n_workers 4
```


### Creación de la malla

Nos desplazaremos a la carpeta:

```bash
cd ./src/seacast_tools
```

Creamos la malla, en este caso la mejor del trabajo:
```bash
python .\create_non_uniform_mesh.py --dataset atlantic --plot 1 --mesh_type fps --sampler fps_weighted --probability_distribution mixed_sigmoid --levels 2 --n_connections 1 --k_neighboors 1 --nodes_amount 394,45
```

En caso de querer usarla, las desplazamos a la carpeta esperada por el modelo:
```bash
python .\move_files.py --graph_type hierarchical --graph fps
```

Tras esto, volvemos al directorio del que partimos:
```bash
cd ../../
```

Más información al respecto puede ser encontrada en la documentación encontrada en: ./docs/mesh_model_modules.md

### Entrenamiento del modelo
Durante el entrenamiento y en la evaluación del modelo, existe la posibilidad de utilizar wandb, siendo esto altamente recomendable para un mejor seguimiento del estado del modelo.

```bash
python train_model.py --dataset atlantic \
                      --n_nodes 1 \
                      --n_workers 4 \
                      --epochs 100 \
                      --lr 0.001 \
                      --batch_size 1 \
                      --step_length 1 \
                      --ar_steps 1 \
                      --optimizer adamw \
                      --scheduler cosine \
                      --processor_layers 4 \
                      --hidden_dim 128 \
                      --model hi_lam \
                      --graph hierarchical \
                      --finetune_start 1 \
                      --custom_run_name fps_mixed_sigmoid_27_9
```

---

### Evaluación del modelo

Una vez entrenado, puedes evaluar el rendimiento del modelo Hi-LAM sobre el conjunto de test ejecutando:

```bash
python train_model.py --dataset atlantic \
                      --data_subset reanalysis \
                      --forcing_prefix forcing \
                      --n_workers 4 \
                      --batch_size 1 \
                      --step_length 1 \
                      --model hi_lam \
                      --graph hierarchical \
                      --processor_layers 4 \
                      --hidden_dim 128 \
                      --n_example_pred 1 \
                      --store_pred 1 \
                      --eval test \
                      --load saved_models/run-20250616_2021121p32judk_fps_weighted_mixed_sigmoid_27_9/last.ckpt
                      -- custom_run_name fps_mixed_sigmoid_27_9
```

Donde cabe destacar que el directorio donde se crearán las predicciones tendrá un nombre gestionado por wandb, por lo que sería recomendarle guardar registro de cada experimento.

