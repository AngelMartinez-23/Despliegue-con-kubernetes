# Despliegue-con-kubernetes

A continuación, se detallan los pasos necesarios para implementar cinco instancias de un servidor desarrollado con FastAPI, todas gestionadas a través de un único servicio.

## Crear aplicación FastAPI
El primer paso es crear un archivo llamado main.py, donde definiremos nuestra API.    

```
    from typing import Optional
    from fastapi import HTTPException # type: ignore
    from fastapi import FastAPI # type: ignore
    from motor.motor_asyncio import AsyncIOMotorClient # type: ignore
    from bson import ObjectId # type: ignore
    from fastapi.encoders import jsonable_encoder # type: ignore
    from pydantic import BaseModel # type: ignore
    import os
    from dotenv import load_dotenv
    from motor.motor_asyncio import AsyncIOMotorClient # type: ignore
    
    load_dotenv()
    app = FastAPI()
    
    # Configuración de la URI
    #MONGO_URI = "mongodb+srv://Grupo1:grupo1@cluster0.h4a3o.mongodb.net/Usuarios?retryWrites=true&w=majority"
    
    MONGO_URI = os.getenv("MONGO_URI")
    
    if not MONGO_URI:
        raise ValueError("Falta la variable de entorno MONGO_URI")
    
    # Crear cliente y base de datos
    client = AsyncIOMotorClient(MONGO_URI)
    db = client["Usuarios"]
    
    class UserSchema(BaseModel):
        id: str
        nombre: str
        password: str
    
    class UserCreateSchema(BaseModel):
        nombre: str
        password: str
    
    class User(BaseModel):
        name: str
        email: str
        age: Optional[int] = None  # Edad opcional
    
    class UserResponse(User):
        id: str  # Convertimos ObjectId a str
    
    # Función para convertir ObjectId a str
    def objectid_to_str(obj):
        if isinstance(obj, ObjectId):
            return str(obj)
        elif isinstance(obj, dict):
            return {k: objectid_to_str(v) for k, v in obj.items()}
        elif isinstance(obj, list):
            return [objectid_to_str(i) for i in obj]
        return obj
    
    @app.get("/")
    async def root():
        return "Bienvenido a la API de usuari"
    
    @app.get("/users")
    async def get_users():
        try:
            # Obtener los usuarios de la colección
            usuarios_collection = db["Usuarios"]
            users = await usuarios_collection.find().to_list(100)
            
            # Convertir ObjectIds a str antes de devolver
            users = [objectid_to_str(user) for user in users]
            
            if not users:
                return {"message": "No hay usuarios en la base de datos"}
            
            return {"usuarios": users}
        
        except Exception as e:
            # Capturar cualquier excepción
            return {"error": f"Hubo un error: {str(e)}"}
        
    @app.get("/users/{user_id}")
    async def get_user(user_id: str):
        user = await db["Usuarios"].find_one({"_id": ObjectId(user_id)})
    
        if not user:
            raise HTTPException(status_code=404, detail="Usuario no encontrado")
    
        user["id"] = str(user["_id"])
        del user["_id"]
    
        return user
    
    @app.post("/users")
    async def create_user(user: UserCreateSchema):
        user_dict = user.dict()  # Convertir Pydantic model a diccionario
        result = await db["Usuarios"].insert_one(user_dict)  # Insertar en MongoDB
        user_dict["_id"] = str(result.inserted_id)  # Convertir ObjectId a string
        return user_dict  # Devolver usuario con _id convertido
    
    @app.put("/users/{user_id}")
    async def update_user(user_id: str, user: UserCreateSchema):
        user_dict = user.dict()  # Convertir el modelo a diccionario
        result = await db["Usuarios"].update_one({"_id": ObjectId(user_id)}, {"$set": user_dict})
    
        if result.matched_count == 0:
            raise HTTPException(status_code=404, detail="Usuario no encontrado")
    
        return {"message": "Usuario actualizado correctamente"}
    
    @app.delete("/users/{user_id}")
    async def delete_user(user_id: str):
        result = await db["Usuarios"].delete_one({"_id": ObjectId(user_id)})
    
        if result.deleted_count == 0:
            raise HTTPException(status_code=404, detail="Usuario no encontrado")
    
        return {"message": "Usuario eliminado correctamente"}
```

## Generar Dockerfile
Dentro del mismo directorio, se debe crear un archivo llamado **Dockerfile** con el siguiente contenido:

```
# Usa una imagen base de Python
FROM python:3.10

# Establece el directorio de trabajo dentro del contenedor
WORKDIR /app

# Instala GPG para manejar el cifrado
RUN apt-get update && apt-get install -y gnupg gpg-agent

# Crear directorio seguro para GPG dentro del contenedor
RUN mkdir -p /root/.gnupg && chmod 700 /root/.gnupg

# Copia los archivos necesarios
COPY .env.gpg ./
COPY private.key ./

# Importa la clave privada sin interacción
RUN gpg --batch --import private.key && rm private.key

# Desencripta el archivo .env.gpg al archivo .env
RUN gpg --batch --yes --decrypt --output .env .env.gpg || echo "Error al desencriptar .env"

# Copia los archivos de la app
COPY requirements.txt ./
COPY . .

# Instala las dependencias
RUN pip install --no-cache-dir -r requirements.txt

# Expone el puerto en el que correrá FastAPI
EXPOSE 8000

# Comando para ejecutar la aplicación
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]

```


Ejecuta el siguiente comando para construir la imagen:"
  
  docker build -t fastapi-secrets .

Imagen 1

## Generar los archivos de configuración para Kubernetes

Crea un archivo YAML llamado *deployment.yaml* con la siguiente configuración:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-deployment
spec:
  replicas: 5  # Definimos 5 réplicas
  selector:
    matchLabels:
      app: fastapi
  template:
    metadata:
      labels:
        app: fastapi
    spec:
      containers:
      - name: fastapi
        image: angelmartinez234/fastapi-secrets # Reemplaza con la imagen que subiste
        ports:
        - containerPort: 8000
```

Genera un archivo YAML llamado *service.yaml* para exponer el servicio:

```
apiVersion: v1
kind: Service
metadata:
  name: fastapi-service
spec:
  selector:
    app: fastapi
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
  type: LoadBalancer  # Para exponer el servicio al exterior

```

## Implementar los archivos en Kubernetes
Si estás utilizando Docker Desktop en Windows, primero debes habilitar Kubernetes en Opciones > Kubernetes.

Imagen2


Para desplegar la aplicación, ejecutamos los siguientes comandos para aplicar la configuración en Kubernetes:

```
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

```

Imagen3

Para verificar que todo está funcionando correctamente y que se muestra en Docker Desktop, ejecuta los siguientes comandos:

```
kubectl get pods         # Ver los pods en ejecución  
kubectl get deployments  # Ver los despliegues  
kubectl get service     # Ver el servicio y su IP externa

```

imagen 6


Por último, se debe mostrar una imagen que visualice todos los archivos de FastAPI creados en Docker Desktop.

imagen 7





