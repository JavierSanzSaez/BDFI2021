# Práctica final BDFI - Javier Sanz Sáez

## Instalando el proyecto
Se clona el proyecto
```
git clone https://github.com/ging/practica_big_data_2019 practicaBDFI
```

Se deben instalar los siguientes componentes (basándose en una distribución Debian Buster):
- Python3 (3.7.3)
- PIP (21.3.1)
```
sudo apt install python3-pip
```
- JDK (1.11.0)
```
sudo apt install default-jdk
```
- SBT (1.5.5)
```
sudo apt-get update
sudo apt-get install apt-transport-https curl gnupg -yqq
echo "deb https://repo.scala-sbt.org/scalasbt/debian all main" | sudo tee /etc/apt/sources.list.d/sbt.list
echo "deb https://repo.scala-sbt.org/scalasbt/debian /" | sudo tee /etc/apt/sources.list.d/sbt_old.list
curl -sL "https://keyserver.ubuntu.com/pks/lookup?op=get&search=0x2EE0EA64E40A89B84B2DF73499E82A75642AC823" | sudo -H gpg --no-default-keyring --keyring gnupg-ring:/etc/apt/trusted.gpg.d/scalasbt-release.gpg --import
sudo chmod 644 /etc/apt/trusted.gpg.d/scalasbt-release.gpg
sudo apt-get update
sudo apt-get install sbt
```
- MongoDB (4.2.15)
```
wget -qO - https://www.mongodb.org/static/pgp/server-4.2.asc | sudo apt-key add -
echo "deb http://repo.mongodb.org/apt/debian buster/mongodb-org/4.2 main" | sudo tee /etc/apt/sources.list.d/mongodb-org-4.2.list
sudo apt-get update
sudo apt-get install -y mongodb-org=4.2.15 mongodb-org-server=4.2.15 mongodb-org-shell=4.2.15 mongodb-org-mongos=4.2.15 mongodb-org-tools=4.2.15
```

- Spark (3.1.2)
```
wget https://www.apache.org/dyn/closer.lua/spark/spark-3.1.2/spark-3.1.2-bin-hadoop3.2.tgz
tar -xvzf spark-3.1.2-bin-hadoop3.2.tgz
mv spark-3.0.0-preview2-bin-hadoop2.7 /opt/spark
```

- Scala (2.11.12-4)

En la fecha de elaboración de esta memoria, la versión disponible en el repositorio de Debian es la versión 2.11.12, la cual satisface los requisitos del proyecto.
```
sudo apt-get install scala
```

- Kafka (2.13-3.0.0)

Se instalará Kafka dentro del proyecto descargado.
```
wget https://www.apache.org/dyn/closer.cgi?path=/kafka/3.0.0/kafka_2.13-3.0.0.tgz
tar -xvzf kafka_2.13-3.0.0.tgz
mv kafka_2.13-3.0.0 practicaBDFI/kafka
```

- Zookeeper

La versión de Kafka tiene Zookeeper incluido, ya que lo requiere, por lo que no es necesario descargarlo de nuevo.

- Paquetes de Python del proyecto
```
cd practicaBDFI
pip install -r requirements.txt
```

## Entrenando el proyecto
Levantamos Zookeeper en una consola
```
cd practicaBDFI
kafka/bin/zookeeper-server-start.sh kafka/config/zookeeper.properties
```
Levantamos el servidor de Kafka en otra consola
```
kafka/bin/kafka-server-start.sh kafka/config/server.properties
```
En otra consola creamos un nuevo topic:
```
    kafka/bin/kafka-topics.sh \
      --create \
      --bootstrap-server localhost:9092 \
      --replication-factor 1 \
      --partitions 1 \
      --topic flight_delay_classification_request

```

Ejecutamos el daemon de mongoDB:
```
systemctl start mongod
systemctl status mongod
```

A continuación nos descargamos los datos de los vuelos y los importamos:
```
./resources/download_data.sh
./resources/import_distances.sh
```

Para entrenar el modelo, necesitamos primero establecer la variable `JAVA_HOME` con el directorio donde tenemos instalado nuestro JDK, así como con la variable `SPARK_HOME`
```
export JAVA_HOME=/usr/lib/jvm/java-1.11.0-openjdk-amd64
export SPARK_HOME=/opt/spark
export PATH=$PATH:$SPARK_HOME/bin:$SPARK_HOME/sbin
```
Ejecutamos el modelo

```
python3 resources/train_spark_mllib_model.py
```
El cual generará ficheros en la carpeta `models`

## Ejecutando el predictor

Primero se debe modificar el fichero Scala en `flight_prediction/src/main/scala/es/upm/dit` y editar la siguiente línea:
```
  val base_path= "/home/user/Desktop/practica_big_data_2019"
```
Sustituyendo el valor por el directorio donde se ha alojado el proyecto.

Una vez guardado el cambio, se ejecuta el proyecto mediante **spark-submit**. Para ello, primero se debe compilar el .jar del proyecto Scala.
```
cd flight_prediction
sbt clean
sbt package
```
Una vez compilado, se ejecuta lo siguiente:

```
spark-submit --class es.upm.dit.ging.predictor.MakePrediction --master local --packages org.mongodb.spark:mongo-spark-connector_2.12:3.0.0,org.apache.spark:spark-sql-kafka-0-10_2.12:3.0.1 <directorio>/practicaBDFI/flight_prediction/target/scala-2.12/flight_prediction_2.12-0.1.jar
```
Sustituyendo `<directorio>` por el directorio padre donde se ha guardado el proyecto. 

## Levantando la aplicación web

Se debe establecer la variable `PROJECT_HOME` con el directorio donde se tiene guardado el proyecto.

```
export PROJECT_HOME=<directorio>/practicaBDFI
```

Finalmente, se ejecuta el fichero Flask

`python3 resources/web/predict_flask.py`

Para acceder a la aplicación web, basta con entrar al navegador web y visitar la dirección http://localhost:5000/flights/delays/predict_kafka

## Usando Docker
Para instalar Docker en el sistema, se ejecutan los siguientes comandos
```
sudo apt update
sudo apt install apt-transport-https ca-certificates curl gnupg2 software-properties-common
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/debian $(lsb_release -cs) stable"
sudo apt update
sudo apt install docker-ce
```
Una vez instalado, levantamos el daemon de Docker
```
systemctl start docker
systemctl status docker
```
Así mismo, se debe instalar Docker-compose para poder orquestar los despliegues de los contenedores

```
sudo curl -L https://github.com/docker/compose/releases/download/1.25.3/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```
Y se comprueba que se ha instalado correctamente a partir de su versión

```
docker-compose --version
```

Una vez instalado Docker y Docker-compose, se debe ir a la carpeta `docker-compose` y ejecutar el siguiente comando:
```
docker-compose up
```

