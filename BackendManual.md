### Backend (Django)

![h](https://cdn.html.it/_4xpBFMBMQUuSbJ1t0E8fxfsoBQ=/480x300/smart/filters:format(webp)/https://www.html.it/app/uploads/2024/09/coding-j.jpeg)

#### Objetivos:

1. Aprender cómo crear un modelo para mensajes.

2. Construir la lógica para que un administrador pueda enviar mensajes a supervisores.

3. Crear una API para listar mensajes y marcarlos como leídos.

4. Hacer que la estructura respete la organización de tu imagen: 
   
                  Repositorio -> Servicio -> Vistas -> Rutas.

---

### Paso 1: Crear el Modelo

- Explicar el propósito del modelo Message.

- Incluir campos básicos:

- title: Título del mensaje.

- content: Contenido del mensaje.

- is_read: Indica si el mensaje ha sido leído.

- receiver: Usuario supervisor al que va dirigido (usaremos ForeignKey a User).

- created_at: Fecha de creación.
  
  **Rutas:**
  
  `microservice_app`  =>  `domain`  => `models` 
  Dentro de esa ruta vamos a crear un archivo llamado `message.py` 
  
  Código Ejemplo:

```python
from django.db import models
from django.contrib.auth.models import User


class Message(models.Model):
    title = models.CharField(max_length=255)
    content = models.TextField()
    is_read = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.title
```

### Paso 2: Crear el Repositorio

- Repositorio: Abstraer consultas a la base de datos.

- Crear una clase MessageRepository que implemente funciones como:

- get_messages_for_user(user)

- mark_message_as_read(message_id)
  
  **Ruta:**
  
  `microservice_app` => `ports` => `repositories` 
  
  Dentro de esa ruta crearemos un archivo llamado `message_repository.py`

Código Ejemplo:

```python
from abc import ABC
from entity_app.domain.models.message import Message

class MessageRepository(ABC):
    def get_all_messages(self):
        return Message.objects.all().order_by('-created_at')

    def mark_message_as_read(self, message_id):
        message = Message.objects.filter(id=message_id).first()
        if message:
            message.is_read = True
            message.save()
        return message
    
    def create_message(self, title, content):
        message = Message(
            title=title,
            content=content,
        )
        message.save()
        return message
```

### Paso 3: Crear el Servicio

- Servicio: Contener la lógica de negocio. Por ejemplo:

- Obtener mensajes no leídos para un usuario.

- Crear un mensaje desde el administrador.
  
  **Ruta:**
  
  `microservice_app` => `domain`  => `services`
  
  Dentro de esa carpteta creamos un archivo llamado `message_service.py`

Código Ejemplo:

```python
from entity_app.ports.repositories.message_repository import MessageRepository

class MessageService:
    def __init__(self, repository: MessageRepository):
        self.repository = repository

    def get_unread_messages(self):
        return self.repository.get_all_messages()

    def create_message(self, title, content):
        return self.repository.create_message(title, content)
    
```

Adicional a eso crearemos nuestro archivo implementador, es muy importante que tengamos este archivo.

```python
from django.apps import apps
from django.contrib.auth.models import User
from entity_app.ports.repositories.message_repository import MessageRepository
from entity_app.domain.models.message import Message


class MessagesImpl(MessageRepository):
    def create_message(self, title, content):
        message = Message(
            title=title,
            content=content,
        )
        message.save()
        return message
    
    def get_all_messages(self):
        return Message.objects.all().order_by('-created_at')
```




### Paso 4: Crear las Vistas

- Vistas (Views): Implementar endpoints API.

- Usar Django REST Framework (DRF) para crear:

- Un endpoint para listar mensajes no leídos (GET).

- Un endpoint para crear mensajes (POST).
  
  **Ruta:**
  
  `microservice_app` => `aplication` => `views`
  
  Dentro de esta carpeta creamos un archivo llamado `messages`

Código Ejemplo:

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated
from django.contrib.auth.models import User
from rest_framework import status
from entity_app.ports.repositories.message_repository import MessageRepository
from entity_app.domain.services.message_service import MessageService
from entity_app.adapters.serializers import MessageSerializer
from entity_app.adapters.impl.messages_impl import MessagesImpl
import logging

logger = logging.getLogger(__name__)


class MessageView(APIView):
    permission_classes = [IsAuthenticated]
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.service = MessageService(MessagesImpl())

    def get(self, request):
        messages = self.service.get_unread_messages()
        serializer = MessageSerializer(messages, many=True)
        return Response(serializer.data)

    def post(self, request, *args, **kwargs):
        data = request.data
        message = self.service.create_message(data['title'], data['content'])
        return Response({'id': message.id}, status=status.HTTP_201_CREATED)
```

### Paso 5: Crear Serializadores

![.](/home/chae/Descargas/image-305.png)

- Serializadores para manejar la conversión entre objetos Python y JSON.
  
  - Creamos las clase `MessageSeralizer`
  
  - Ruta en el microservicio: `nombre_mircroservicio_app` => `adapters`=> `seralizers.py`

Código Ejemplo:

```python
from rest_framework import serializers
from .models import Message

# otro codigo ya escrito
#....

# Al final agregamos nuestra linea no olvidarse de importar el modelo
class MessageSerializer(serializers.ModelSerializer):
    class Meta:
        model = Message
        fields = ['id', 'title', 'content', 'is_read', 'created_at']
```

### Paso 6: Definir las Rutas

- Configurar rutas para el API.

Código Ejemplo:

```python
from django.urls import path
from .views import MessageView

urlpatterns = [
    path('messages/', MessageView.as_view(), name='messages'),
]
```
