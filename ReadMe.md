# Sistema de Recomendación de Contenido Netflix en AWS

## Descripción General

Este proyecto implementa un sistema de recomendación de contenido para Netflix utilizando datos de TMDB. El sistema utiliza técnicas de procesamiento de lenguaje natural y aprendizaje automático para generar recomendaciones personalizadas basadas en la similitud de contenido.

El sistema admite dos métodos de recomendación:
1. **Recomendación basada en una sola variable**: Utiliza solo la descripción (overview) del contenido
2. **Recomendación basada en múltiples variables**: Combina título, descripción, director y elenco

## Arquitectura

El sistema se implementa completamente en AWS, utilizando una arquitectura serverless y microservicios. El siguiente diagrama muestra los principales componentes y sus interacciones:

### Diagrama de Arquitectura

El diagrama a continuación ilustra cómo los diferentes servicios de AWS trabajan juntos en esta solución:

![AWS arquitectura para Netflix](/images/aws_architecture.png)

### Componentes Principales:

- **Almacenamiento**: Amazon S3, Amazon DynamoDB
- **Procesamiento**: AWS Lambda, AWS Glue, Amazon SageMaker
- **Orquestación**: AWS Step Functions, Amazon EventBridge
- **API**: Amazon API Gateway
- **Frontend**: Aplicación Streamlit en AWS Elastic Beanstalk
- **Monitorización**: Amazon CloudWatch

## Requisitos

- Cuenta de AWS
- API Key de TMDB (https://www.themoviedb.org/documentation/api)
- AWS CLI configurado
- Python 3.8 o superior
- pip y virtualenv

## Estructura del Proyecto

```
netflix-recommendation/
├── cloudformation/
│   └── template.yaml         # Plantilla CloudFormation para infraestructura
├── lambda/
│   ├── etl/                  # Función Lambda para ETL
│   │   ├── index.py
│   │   └── requirements.txt
│   └── recommendation-api/   # Función Lambda para API de recomendaciones
│       ├── index.py
│       └── requirements.txt
├── streamlit/                # Aplicación Streamlit
│   ├── app.py
│   └── requirements.txt
├── notebooks/                # Jupyter notebooks para análisis
│   └── recommendation_system.ipynb
├── scripts/                  # Scripts de utilidad
│   └── deploy.sh             # Script para despliegue
└── README.md                 # Este archivo
```

## Guía de Despliegue

### 1. Clonar el Repositorio

```bash
git clone [URL-DEL-REPOSITORIO]
cd netflix-recommendation
```

### 2. Configurar Variables de Entorno

Cree un archivo `.env` en la raíz del proyecto con las siguientes variables:

```
TMDB_API_KEY=su_api_key_de_tmdb
AWS_REGION=us-east-1  # O la región de su preferencia
ENVIRONMENT=dev        # dev, test, o prod
ADMIN_EMAIL=su_email@ejemplo.com
```

### 3. Desplegar la Infraestructura

Utilizando AWS CloudFormation:

```bash
# Instalar dependencias
pip install -r requirements.txt

# Desplegar usando el script
./scripts/deploy.sh

# Alternativamente, usar AWS CLI directamente
aws cloudformation create-stack \
  --stack-name netflix-recommendation-system \
  --template-body file://cloudformation/template.yaml \
  --parameters ParameterKey=Environment,ParameterValue=$ENVIRONMENT \
               ParameterKey=TMDBApiKey,ParameterValue=$TMDB_API_KEY \
               ParameterKey=AdminEmail,ParameterValue=$ADMIN_EMAIL \
  --capabilities CAPABILITY_IAM
```

### 4. Subir Dataset Inicial a S3

```bash
# Crear la estructura de directorios en S3
aws s3 mb s3://netflix-recommendation-data-$AWS_ACCOUNT_ID-$ENVIRONMENT

# Subir el dataset
aws s3 cp netflix_all_data.csv s3://netflix-recommendation-data-$AWS_ACCOUNT_ID-$ENVIRONMENT/raw/netflix_all_data.csv
```

### 5. Iniciar el Pipeline ETL

```bash
# Ejecutar manualmente el Step Function de ETL para el procesamiento inicial
aws stepfunctions start-execution \
  --state-machine-arn arn:aws:states:$AWS_REGION:$AWS_ACCOUNT_ID:stateMachine:netflix-etl-workflow-$ENVIRONMENT
```

### 6. Desplegar la Aplicación Streamlit

```bash
# Navegar al directorio de la aplicación Streamlit
cd streamlit

# Crear archivo zip para Elastic Beanstalk
zip -r ../streamlit-app.zip .

# Subir a S3
aws s3 cp ../streamlit-app.zip s3://netflix-recommendation-data-$AWS_ACCOUNT_ID-$ENVIRONMENT/deployment/

# Crear versión de aplicación
aws elasticbeanstalk create-application-version \
  --application-name netflix-recommendation-app-$ENVIRONMENT \
  --version-label v1 \
  --source-bundle S3Bucket=netflix-recommendation-data-$AWS_ACCOUNT_ID-$ENVIRONMENT,S3Key=deployment/streamlit-app.zip

# Desplegar versión
aws elasticbeanstalk update-environment \
  --environment-name netflix-recommendation-env-$ENVIRONMENT \
  --version-label v1
```

### 7. Verificar el Despliegue

Una vez completado el despliegue, se puede acceder a la aplicación Streamlit a través de la URL proporcionada en las salidas de CloudFormation:

```bash
aws cloudformation describe-stacks \
  --stack-name netflix-recommendation-system \
  --query "Stacks[0].Outputs[?OutputKey=='StreamlitURL'].OutputValue" \
  --output text
```

## Uso del Sistema

### Interfaz de Streamlit

La interfaz de usuario Streamlit permite:
- Buscar películas y series en el catálogo de Netflix
- Filtrar por tipo de contenido (películas/series) y género
- Obtener recomendaciones personalizadas
- Ver detalles de cada recomendación

### API de Recomendaciones

La API proporciona endpoints para integrar las recomendaciones en otras aplicaciones:

```
GET /recommendations/{title_id}?feature_type=multi&count=10
```

Parámetros:
- `title_id`: ID del título para el que se desean recomendaciones
- `feature_type`: Método de recomendación (`overview` o `multi`)
- `count`: Número de recomendaciones a devolver (predeterminado: 10)

## Notebook de Análisis

El notebook `recommendation_system.ipynb` proporciona un análisis detallado del sistema de recomendación, incluyendo:
- Exploración de datos
- Preprocesamiento de texto
- Implementación de TF-IDF y similitud de coseno
- Evaluación de resultados
- Ejemplos de recomendaciones

## Mantenimiento y Actualización

### Actualización de Datos

Los datos se actualizan diariamente a través del pipeline ETL programado. Para forzar una actualización manual:

```bash
aws lambda invoke \
  --function-name netflix-etl-$ENVIRONMENT \
  --payload '{"update_from_tmdb": true}' \
  response.json
```

### Monitorización

El sistema configura automáticamente alarmas de CloudWatch para:
- Latencia de API elevada
- Errores 5xx en la API
- Fallos en el pipeline ETL

Las notificaciones se envían al correo electrónico del administrador configurado.

## Solución de Problemas

### Logs de Lambda

```bash
aws logs filter-log-events \
  --log-group-name /aws/lambda/netflix-etl-$ENVIRONMENT \
  --filter-pattern "ERROR"
```

### Verificar Estado de Step Functions

```bash
aws stepfunctions list-executions \
  --state-machine-arn arn:aws:states:$AWS_REGION:$AWS_ACCOUNT_ID:stateMachine:netflix-etl-workflow-$ENVIRONMENT \
  --max-items 5
```

## Licencia

Este proyecto está licenciado bajo la Licencia MIT - vea el archivo LICENSE para más detalles.

## Contacto

Para cualquier pregunta o sugerencia, póngase en contacto con el equipo de desarrollo.
