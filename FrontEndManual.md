### FrontEnd (React)

![a](https://media2.dev.to/dynamic/image/width=1000,height=420,fit=cover,gravity=auto,format=auto/https%3A%2F%2Fdev-to-uploads.s3.amazonaws.com%2Fuploads%2Farticles%2F2tyembrdqzbcnkqc9bqv.png)

#### Objetivos:

1. Crear una interfaz donde el usuario **supervisor** pueda ver mensajes no leídos.

2. Implementar una campanita (notificación) para indicar mensajes pendientes.

3. Diseñar un formulario para que el **admin** pueda enviar mensajes.

4. Usar la estructura de componentes según estructura: 
   
       Caso de uso -> Servicio -> Módulo API -> Interface -> Container.

### **Paso 1: Configurar la API**

- Crear un módulo centralizado para gestionar las peticiones a la API (`API module`).
- Usar **Axios** para hacer las solicitudes HTTP.

**Código Ejemplo - `src/infraestructure/api/Message/Messages.ts`:**

```ts
import { AxiosError, AxiosInstance } from "axios";
import { AxiosError, AxiosInstance } from "axios";
import { TRANSPARENCY_PATH } from "..";

class MessageApi {
    constructor(private api: AxiosInstance) { }
    // Obtener mensajes
    getMessages = async () => {
        try {
            const response = await this.api.get(TRANSPARENCY_PATH+'/messages/');
            return response.data;
        } catch (error) {
            if (error instanceof AxiosError) {

                const message = error?.response?.data?.message || "Ocurrió un error al obtener los mensajes"
                throw new Error(message);
            } else {
                throw new Error("Ocurrió un error al obtener los mensajes");

            }
        }
    };

    // Crear mensaje
    createMessage = async (title: string, content: string) => {
        try {
            const response = await this.api.post(TRANSPARENCY_PATH + '/messages/', {title, content});
            return response.data;
        } catch (error) {
            if (error instanceof AxiosError) {
                const message = error?.response?.data?.message || "Ocurrió un error al crear el mensaje"
                throw new Error(message);
            } else {
                throw new Error("Ocurrió un error al crear el mensaje");

            }
        }
    };
}

export default MessageApi;
```

**Código Ejemplo - `src/infraestructure/api/Message/interface.ts`:**

```ts
import { BaseObject } from "..";

export interface MessageDTO extends BaseObject {
    id: number;
    title: string;
    content: string;
    is_read: boolean;
    created_at: string;
}
```

### **Paso 2: Implementar el Servicio**

- **Servicio**: Proveer lógica reutilizable que interactúe con el API.
- Conectar este servicio con el **Caso de uso** en la app.

**Código Ejemplo - `src/infraestructure/service/MessageService.ts`:**

```ts
import MessageApi from '../Api/Messages/Message.ts';

class MessageService {
    constructor(private readonly api: MessageApi) { }
    
    async fetchMessages() {
      return await this.api.getMessages();
    }
  
    async sendMessage(title: string, content: string) {
      return await this.api.createMessage(title, content);
    }
 }

export default MessageService
```

### **Paso 3: Crear el Caso de Uso**

- **Caso de uso**: Gestionar la lógica para:
  - Consultar mensajes no leídos.
  - Renderizar mensajes en la interfaz.
  - Disparar acciones como "marcar mensaje como leído".

**Código Ejemplo - `src/domain/useCases/useCases/MessageUseCase/MessageUseCase.ts`:**

```ts
import MessageService from '../../../infrastructure/Services/MessageService';

class MessageUseCase {
    constructor(
      private service: MessageService,
    ) { }

  async getUnreadMessages() {
    return await this.service.fetchMessages();
  }

  async createNewMessage(title: string, content: string) {
    return await this.service.sendMessage(title, content);
  }
}

export default MessageUseCase
```

### **Paso 4: Crear el Presenter**

- El Presenter maneja el estado y la lógica del componente.
- Aquí es donde se maneja el flujo de datos entre el `Container` y el `Caso de uso`.

**Código Ejemplo - `presenters/MessagePresenter.js`:**

```tsx
import { FormEvent } from "react";

interface Props {
    handleSubmit: (e: FormEvent<HTMLFormElement>) => void;
    title: string;
    content: string;
    //receiverId: string;
    setTitle: (value: string) => void;
    setContent: (value: string) => void;
    //setReceiverId: (value: string) => void;
}

const InboxCreatePresenter = ({
    handleSubmit,
    title,
    content,
    //receiverId,
    setTitle,
    setContent,
    //setReceiverId,
}: Props) => {
    return (
        <div className="flex flex-col justify-center items-center">
            <h2 className="text-3xl">Enviar mensajes a Supervisores</h2>
            <div className="w-full max-w-xs mt-10">
                <form
                    onSubmit={handleSubmit}
                    className="bg-white shadow-md rounded px-8 pt-6 pb-8 mb-4 flex flex-col"
                >
                    <div className="mb-4">
                        <label className="block text-gray-700 text-sm font-bold mb-2">
                            Titulo
                        </label>
                        <input
                            className="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
                            type="text"
                            placeholder="Titulo"
                            value={title}
                            onChange={(e) => setTitle(e.target.value)}
                        />
                    </div>
                    <div className="mb-4">
                        <label className="block text-gray-700 text-sm font-bold mb-2">
                            Contenido
                        </label>
                        <textarea
                            className="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
                            placeholder="Contenido"
                            value={content}
                            onChange={(e) => setContent(e.target.value)}
                        />
                    </div>
                    <button
                        className="bg-gray-300 hover:bg-gray-400 text-gray-800 font-bold py-2 px-4 rounded inline-flex text-center"
                        type="submit"
                    >
                        Enviar Mensaje
                    </button>
                </form>
            </div>
        </div>
    );
};

export default InboxCreatePresenter;
```

### **Paso 5: Crear el Container**

- **Container**: Conecta el Presenter con los componentes visuales.
- Contiene la lógica para mostrar u ocultar el listado de mensajes o el formulario.

**Código Ejemplo - `containers/MessageContainer.js`:**

```tsx
import InboxCreatePresenter from "./InboxCreatePresenter";
import MessageUseCase from "../../../../domain/useCases/MessagesUseCase/MessagesUseCase";
import { FormEvent, useState } from "react";

function InboxCreateContainer({
    usecase
}:{
    usecase: MessageUseCase
}) {
    const [title, setTitle] = useState('');
    const [content, setContent] = useState('');
    //const [receiverId, setReceiverId] = useState('');


    const handleSubmit = (e: FormEvent<HTMLFormElement>) => {
        e.preventDefault();
        console.log(MessageUseCase)
        usecase.createNewMessage(title, content);
        setTitle('');
        setContent('');
        //setReceiverId('');
    };

    return (
        <InboxCreatePresenter
            handleSubmit={handleSubmit}
            title={title}
            content={content}
            //receiverId={receiverId}
            setTitle={setTitle}
            setContent={setContent}
            //setReceiverId={setReceiverId}
        />
    );
}

export default InboxCreateContainer;
```

Adicional a esto debemos asegurarnos de tener estos archivos:

Ruta: `src/interfaces/web/Admin/Inbox/index.tsx`

```tsx
import InboxCreateContainer from "../../../../components/Admin/Inbox/create/InboxCreateContainer";
import MessageUseCase from "../../../../domain/useCases/MessagesUseCase/MessagesUseCase";
import api from "../../../../infrastructure/Api";
import MessageApi from "../../../../infrastructure/Api/Messages/Message";
import MessageService from "../../../../infrastructure/Services/MessageService";

function InboxCreate() {
    // Crear una instancia para api
    const messageApiInstance = new MessageApi(api)
    // Crear una instancia de MessageService
    const messageServiceInstance = new MessageService(messageApiInstance);
    // Pasar el servicio al constructor de MessageUseCase
    const messageUseCaseInstance = new MessageUseCase(messageServiceInstance);
    return (
        <>
            <InboxCreateContainer usecase={messageUseCaseInstance} />
        </>
    );
}

export default InboxCreate;
```

Ruta: `src/utils/menu.tsx`

```tsx
{
    name: 'Mensajes Masivos',
    path: '/admin/inbox/create',
    visible: true,
    icon: <GrCompliance size={25} className="text-slate-500" />,
    permission_required: 'view_establishment',
    element: <InboxCreate />
  },
```



### **Paso 6: Crear la Interfaz**

- Crear componentes funcionales básicos para:
  - **Lista de mensajes**.`
  - **Formulario de envío**.
  - **Campanita de notificaciones**.

**Código Ejemplo - `src/components/Admin/Notifications/Inbox_container.tsx`:**

```tsx
import { useEffect, useState } from 'react';
import MessageUseCase from '../../../domain/useCases/MessagesUseCase/MessagesUseCase';


type Message = {
  id: string;
  title: string;
  content: string;
};

const MessageList = ({
  usecase
}:{
  usecase: MessageUseCase
}) => {
  const [messages, setMessages] = useState<Message[]>([]);
  const [loading, setLoading] = useState<boolean>(true);

  useEffect(() => {
    const fetchMessages = async () => {
      setLoading(true);
      try {
        const data = await usecase.getUnreadMessages();
        setMessages(data); // Asegúrate de que `data` tenga el formato correcto
      } catch (error) {
        console.error('Error fetching messages:', error);
      } finally {
        setLoading(false);
      }
    };

    fetchMessages();
  }, []);

  if (loading) return <p>Loading...</p>;

  return (
    <ul className="space-y-4 bg-gray-50 p-6 rounded-lg shadow-md">
      {messages.map((msg: any) => (
        <li
          key={msg.id}
          className="bg-white p-4 rounded-lg shadow hover:shadow-lg transition-shadow border border-gray-200"
        >
          <h3 className="text-lg font-semibold text-gray-800 mb-2">{msg.title}</h3>
          <p className="text-gray-600 text-sm">{msg.content}</p>
          <p className="text-gray-400 text-xs mt-4 text-right">
            {new Date(msg.created_at).toLocaleDateString('es-ES', {
              year: 'numeric',
              month: 'long',
              day: 'numeric',
            })}
          </p>
        </li>
      ))}
    </ul>
  );
};

export default MessageList;
```

**Código Ejemplo - `src/interfaces/web/Transparency/Notifications/index.tsx`:

- useState:  

```tsx
import MessageList from "../../../../components/Admin/Notifications/Inbox_container";
import MessageUseCase from "../../../../domain/useCases/MessagesUseCase/MessagesUseCase";
import api from "../../../../infrastructure/Api";
import MessageApi from "../../../../infrastructure/Api/Messages/Message";
import MessageService from "../../../../infrastructure/Services/MessageService";



const VerNotificaciones = () => {
    // Crear una instancia para api
    const messageApiInstance = new MessageApi(api)

    // Crear una instancia de MessageService
    const messageServiceInstance = new MessageService(messageApiInstance);

    // Pasar el servicio al constructor de MessageUseCase
    const messageUseCaseInstance = new MessageUseCase(messageServiceInstance);
  return (
    <>
        <MessageList usecase={messageUseCaseInstance} />
    </>
  )
}

export default VerNotificaciones
```

Código Ejemplo - `src/components/Common/HeaderPages/index.tsx`:

```tsx
<Link to='/admin/notifications' className="group relative inline-block cursor-pointer rounded-t-md p-2 text-lg transition hover:bg-primary/20 w-[110px] text-center">
    <span className="text-pretty text-base font-medium">Avisos 🔔</span>
    <span className="absolute -bottom-1 left-1/2 h-0.5 w-0 bg-primary transition-all group-hover:w-3/6"></span>
    <span className="absolute -bottom-1 right-1/2 h-0.5 w-0 bg-primary transition-all group-hover:w-3/6"></span>
</Link>
```
