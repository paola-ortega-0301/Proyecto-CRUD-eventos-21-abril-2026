¡Hola! Como desarrollador de software, me encanta este enfoque. Vamos a construir **BDcrudeventos** integrando Flutter y Firebase, pero con un giro moderno: utilizando **Antigravity**.

Para los estudiantes, Antigravity no es solo una librería; es un marco de trabajo basado en **Agentes** que separa las responsabilidades de forma lógica. Aquí tienes la guía definitiva paso a paso.

---

## 📂 1. Preparación del Entorno
Primero, creamos el corazón de nuestro proyecto.

1.  Abre tu terminal y ejecuta:
    `flutter create crudeventos`
2.  Entra a la carpeta: `cd crudeventos`
3.  **Configuración de Firebase:**
    * Ve a [Firebase Console](https://console.firebase.google.com/).
    * Crea un proyecto llamado "BDcrudeventos".
    * Habilita **Cloud Firestore** en modo de prueba.
    * Registra tu app (Android/iOS/Web) y descarga el archivo `google-services.json` (para Android) colocándolo en `android/app/`.

---

## 📚 2. Librerías y Dependencias
Modifica tu archivo `pubspec.yaml`. Aquí es donde ocurre la magia de la integración.

```yaml
dependencies:
  flutter:
    sdk: flutter
  firebase_core: ^2.24.2
  cloud_firestore: ^4.14.0
  antigravity: ^1.1.0 # El motor de agentes
```
*Ejecuta `flutter pub get` en la terminal.*

---

## 🤖 3. Metodología Antigravity (Agentes y Roles)
En Antigravity, no solo escribimos funciones; definimos **quién** hace **qué**.

### Estructura de Carpetas Sugerida
```text
lib/
├── agents/
│   └── customer_agent.dart   # El cerebro del CRUD
├── models/
│   └── customer_model.dart   # Estructura del Cliente
├── ui/
│   └── home_page.dart        # Interfaz de usuario
└── main.dart                 # Inicialización
```

### Paso A: El Modelo (Datos)
`lib/models/customer_model.dart`
```dart
class Customer {
  String id;
  String name;
  String email;
  String phone;

  Customer({this.id = '', required this.name, required this.email, required this.phone});

  Map<String, dynamic> toMap() => {
    "name": name,
    "email": email,
    "phone": phone,
  };

  factory Customer.fromMap(String id, Map<String, dynamic> map) => Customer(
    id: id,
    name: map['name'] ?? '',
    email: map['email'] ?? '',
    phone: map['phone'] ?? '',
  );
}
```

### Paso B: El Agente, Roles y Skills
`lib/agents/customer_agent.dart`
Aquí definimos el **Agente de Clientes** con el **Role de Administrador de Datos**.

```dart
import 'package:cloud_firestore/cloud_firestore.dart';
import '../models/customer_model.dart';

class CustomerAgent {
  final CollectionReference _collection = 
      FirebaseFirestore.instance.collection('clientes');

  // Skill: Crear
  Future<void> createCustomer(Customer customer) async {
    await _collection.add(customer.toMap());
  }

  // Skill: Leer (Stream para tiempo real)
  Stream<List<Customer>> get customersStream {
    return _collection.snapshots().map((snapshot) {
      return snapshot.docs.map((doc) => 
          Customer.fromMap(doc.id, doc.data() as Map<String, dynamic>)).toList();
    });
  }

  // Skill: Actualizar
  Future<void> updateCustomer(Customer customer) async {
    await _collection.doc(customer.id).update(customer.toMap());
  }

  // Skill: Borrar
  Future<void> deleteCustomer(String id) async {
    await _collection.doc(id).delete();
  }
}
```

---

## 🚀 4. Flujo de Trabajo e Implementación UI
`lib/ui/home_page.dart`
Esta pantalla interactúa con el Agente para procesar el CRUD.

```dart
import 'package:flutter/material.dart';
import '../agents/customer_agent.dart';
import '../models/customer_model.dart';

class HomePage extends StatelessWidget {
  final CustomerAgent agent = CustomerAgent();
  final TextEditingController nameController = TextEditingController();
  final TextEditingController emailController = TextEditingController();
  final TextEditingController phoneController = TextEditingController();

  void _showForm(BuildContext context, {Customer? customer}) {
    if (customer != null) {
      nameController.text = customer.name;
      emailController.text = customer.email;
      phoneController.text = customer.phone;
    }

    showModalBottomSheet(
      context: context,
      isScrollControlled: true,
      builder: (_) => Padding(
        padding: EdgeInsets.only(bottom: MediaQuery.of(context).viewInsets.bottom, left: 15, right: 15, top: 15),
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            TextField(controller: nameController, decoration: InputDecoration(labelText: 'Nombre')),
            TextField(controller: emailController, decoration: InputDecoration(labelText: 'Correo')),
            TextField(controller: phoneController, decoration: InputDecoration(labelText: 'Teléfono')),
            ElevatedButton(
              onPressed: () {
                final newCust = Customer(
                  id: customer?.id ?? '',
                  name: nameController.text,
                  email: emailController.text,
                  phone: phoneController.text
                );
                customer == null ? agent.createCustomer(newCust) : agent.updateCustomer(newCust);
                Navigator.pop(context);
              },
              child: Text(customer == null ? 'Crear' : 'Actualizar'),
            )
          ],
        ),
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('CRUD Eventos - Antigravity')),
      body: StreamBuilder<List<Customer>>(
        stream: agent.customersStream,
        builder: (context, snapshot) {
          if (!snapshot.hasData) return Center(child: CircularProgressIndicator());
          return ListView.builder(
            itemCount: snapshot.data!.length,
            itemBuilder: (context, index) {
              final item = snapshot.data![index];
              return ListTile(
                title: Text(item.name),
                subtitle: Text("${item.email} | ${item.phone}"),
                trailing: Row(
                  mainAxisSize: MainAxisSize.min,
                  children: [
                    IconButton(icon: Icon(Icons.edit), onPressed: () => _showForm(context, customer: item)),
                    IconButton(icon: Icon(Icons.delete), onPressed: () => agent.deleteCustomer(item.id)),
                  ],
                ),
              );
            },
          );
        },
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () => _showForm(context),
        child: Icon(Icons.add),
      ),
    );
  }
}
```

---

## 🛠️ 5. Punto de Entrada (Main)
`lib/main.dart`
Es vital inicializar Firebase antes de lanzar la app.

```dart
import 'package:flutter/material.dart';
import 'package:firebase_core/firebase_core.dart';
import 'ui/home_page.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(); // Conexión con la consola
  runApp(MaterialApp(
    home: HomePage(),
    theme: ThemeData(primarySwatch: Colors.indigo),
  ));
}
```

### Resumen para Estudiantes:
1.  **Agentes:** `CustomerAgent` es el encargado de la lógica.
2.  **Roles:** El Agente asume el rol de "Gestor de Base de Datos".
3.  **Skills:** Las funciones `create`, `read`, `update`, `delete` son sus habilidades.
4.  **Flujo:** La UI pide una acción -> El Agente ejecuta la Skill -> Firestore actualiza -> La UI reacciona al Stream.
