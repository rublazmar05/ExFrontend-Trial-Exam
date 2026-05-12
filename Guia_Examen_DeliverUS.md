# Guía de Resolución Definitiva: Examen DeliverUS - React Native

Esta guía detalla, paso a paso y con fragmentos de código exactos, cómo resolver los problemas típicos que vas a encontrar en el examen (basándonos en la resolución completa del examen de Pedidos). Úsala como "chuleta" para no olvidar configuraciones críticas.

---

## FASE 0: Antes de empezar (Reglas de Oro)
1. **Identifica las entidades y pantallas:** Mira qué pantallas te piden tocar y localízalas en `src/screens`.
2. **Revisa la API (`src/api/`):** Mira qué endpoints te han dado y cuáles tienes que completar. Fíjate bien en la URL. Ej:
   ```javascript
   function getRestaurantOrders (restaurantId) {
     return get(`restaurants/${restaurantId}/orders`)
   }
   function getRestaurantAnalytics (restaurantId) {
     // SI TE FALTA UNO, SE CREA ASÍ:
     return get(`restaurants/${restaurantId}/analytics`)
   }
   ```
3. **Navegación (`src/screens/...Stack.js`):** Si te piden crear una pantalla (ej. `EditOrderScreen`), **tienes que registrarla**.
   ```javascript
   import EditOrderScreen from '../orders/EditOrderScreen'
   // Dentro de Stack.Navigator:
   <Stack.Screen
     name='EditOrderScreen'
     component={EditOrderScreen}
     options={{ title: 'Edit order' }} 
   />
   ```

---

## TAREA 1: Obtener datos de la API y mostrar un Listado (`FlatList`)

**Objetivo:** Pedir datos al backend y pintarlos en una lista en pantalla (ej. `OrdersScreen`).

### 1. Variables de Estado (`useState`)
Necesitas estado para la lista de elementos y, si te piden métricas, otro estado.
```javascript
const [orders, setOrders] = useState([])
const [analytics, setAnalytics] = useState(null) // null si es un objeto que tarda en cargar
```

### 2. Pedir Datos a la API (`useEffect` y funciones `fetch`)
Haz peticiones asíncronas atrapando errores con `try...catch`. **Llama a estas funciones desde un `useEffect`**.
```javascript
useEffect(() => {
  fetchRestaurantDetail()
  fetchRestaurantOrders()
  fetchRestaurantAnalytics()
}, [route])

const fetchRestaurantOrders = async () => {
  try {
    const fetchedOrders = await getRestaurantOrders(route.params.id) // Fíjate de dónde sacas el ID
    setOrders(fetchedOrders)
  } catch (error) {
    showMessage({
      message: `Error retrieving orders: ${error}`,
      type: 'error',
      style: GlobalStyles.flashStyle,
      titleStyle: GlobalStyles.flashTextStyle
    })
  }
}
```

### 3. Pintar el listado (`FlatList`)
Ve al final del componente, al `return`, y monta el `FlatList`.
```javascript
return (
  <FlatList
    style={styles.container}
    data={orders}
    renderItem={renderOrder} // La función que pinta cada fila
    keyExtractor={item => item.id.toString()} // Obligatorio
    ListHeaderComponent={renderHeader} // Opcional, si hay cabecera
    ListEmptyComponent={renderEmptyOrdersList} // Opcional, si no hay datos
  />
)
```

### 4. Diseñar cada elemento (`renderItem`)
Usa componentes de UI (Text, View, ImageCard).
```javascript
const renderOrder = ({ item }) => {
  return (
    <ImageCard
      imageUri={getOrderImage(item.status)}
      title={`Order created at ${item.createdAt}`}
    >
      <TextRegular>Status: {item.status}</TextRegular>
      <TextRegular>Address: {item.address}</TextRegular>
      <TextSemiBold>{item.price.toFixed(2)}€</TextSemiBold>
      
      {/* AQUÍ IRÁN LOS BOTONES LUEGO */}
    </ImageCard>
  )
}
```

---

## TAREA 2: Botones, Renderizado Condicional y Acciones a la API

**Objetivo:** Añadir botones que hacen algo (como navegar o llamar a un endpoint de cambio de estado).

### 1. Botón de Navegación (Ej: Editar)
Si vas a otra pantalla, debes pasar el ID del elemento pulsado en los parámetros de la ruta.
```javascript
<Pressable
  onPress={() => navigation.navigate('EditOrderScreen', { orderId: item.id })} // ¡ATENCIÓN A CÓMO LLAMAS AL PARÁMETRO!
  style={({ pressed }) => [
    { backgroundColor: pressed ? GlobalStyles.brandBlueTap : GlobalStyles.brandBlue },
    styles.actionButton
  ]}>
  <View style={{ flex: 1, flexDirection: 'row', justifyContent: 'center' }}>
    <MaterialCommunityIcons name='pencil' color={'white'} size={20}/>
    <TextRegular textStyle={styles.text}>Edit</TextRegular>
  </View>
</Pressable>
```

### 2. Botón Condicional con Acción de API (Ej: Avanzar Estado)
Si el botón NO debe verse en cierto estado (ej. `delivered`), envuélvelo en una condición (`{ condicion && <Elemento/> }`).
```javascript
{item.status !== 'delivered' &&
  <Pressable
    onPress={() => handleNextStatus(item)}
    style={({ pressed }) => [
      { backgroundColor: pressed ? GlobalStyles.brandGreenTap : GlobalStyles.brandGreen },
      styles.actionButton
    ]}>
    <View style={{ flex: 1, flexDirection: 'row', justifyContent: 'center' }}>
      <MaterialCommunityIcons name='skip-next' color={'white'} size={20}/>
      <TextRegular textStyle={styles.text}>Advance</TextRegular>
    </View>
  </Pressable>
}
```

### 3. Función que llama a la API y refresca la vista
Si cambias un dato en la API, **tienes que volver a llamar a las funciones fetch** para que la pantalla se actualice.
```javascript
const handleNextStatus = async (order) => {
  try {
    await nextStatus(order)
    showMessage({ message: `Updated`, type: 'success', style: GlobalStyles.flashStyle, titleStyle: GlobalStyles.flashTextStyle })
    
    // REFRESCAR LOS DATOS DE LA PANTALLA
    fetchRestaurantOrders() 
    fetchRestaurantAnalytics()
  } catch (error) {
    showMessage({ message: `Error: ${error}`, type: 'danger', /* estilos */ })
  }
}
```

---

## TAREA 3: Formularios con Edición y Validación (`Formik` + `Yup`)

**Objetivo:** Pantalla para editar datos, con reglas obligatorias y pre-rellenada con los datos actuales de la API.

### 1. Variables de Estado y Esquema de Validación
```javascript
import * as yup from 'yup'
import { Formik } from 'formik'

// Fuera del return:
const [order, setOrder] = useState({})
const [initialValues, setInitialValues] = useState({ address: '', price: '' })
const [backendErrors, setBackendErrors] = useState([])

// Reglas YUP
const validationSchema = yup.object().shape({
  address: yup.string().required('Address is required'),
  price: yup.number().required('Price is required').moreThan(0)
})
```

### 2. Recuperar Datos Iniciales (`useEffect`)
Usa el ID que pasaste por navegación (`route.params.orderId`) para traer los datos del objeto. **ACTUALIZA EL ESTADO DE `initialValues`**.
```javascript
useEffect(() => {
  async function fetchOrder () {
    try {
      const fetchedOrder = await getById(route.params.orderId)
      setOrder(fetchedOrder)
      // MUY IMPORTANTE: Preparar datos para Formik. Convertir números a strings si es un input de texto.
      setInitialValues({
        address: fetchedOrder.address,
        price: fetchedOrder.price.toString() 
      })
    } catch (error) {
      // Manejar error...
    }
  }
  fetchOrder()
}, [route.params.orderId]) // Dependencia muy importante
```

### 3. Guardar Datos (`onSubmit`)
```javascript
const updateOrder = async (values) => {
  setBackendErrors([])
  try {
    await update(order.id, values) // update(id_del_elemento, valores_del_formulario)
    showMessage({ message: `Success`, type: 'success', /* estilos */ })
    // Vuelve atrás. (Pasa dirty: true si la pantalla de lista requiere recargar)
    navigation.navigate('OrdersScreen', { id: order.restaurantId, dirty: true }) 
  } catch (error) {
    setBackendErrors(error.errors || [])
  }
}
```

### 4. Renderizar Formulario (`<Formik>`)
**CLAVE CRÍTICA:** Usa `enableReinitialize` para que cuando el `useEffect` traiga los datos de la API, el formulario se actualice.
```javascript
return (
  <Formik
    enableReinitialize // IMPRESCINDIBLE PARA EDICIONES
    validationSchema={validationSchema}
    initialValues={initialValues}
    onSubmit={updateOrder}
  >
    {({ handleSubmit }) => (
      <ScrollView>
        <View style={{ alignItems: 'center' }}>
          <View style={{ width: '60%' }}>
            
            {/* CAMPOS DE TEXTO */}
            <InputItem name="address" label="Address:" />
            <InputItem name="price" label="Price:" keyboardType="numeric" />

            {/* MOSTRAR ERRORES DEL BACKEND */}
            {backendErrors.length > 0 &&
              backendErrors.map((error, index) => (
                <TextError key={index}>{error.param} - {error.msg}</TextError>
              ))}

            {/* BOTÓN DE GUARDAR */}
            <Pressable onPress={handleSubmit} style={styles.button}>
              <View style={{ flex: 1, flexDirection: 'row', justifyContent: 'center' }}>
                 <MaterialCommunityIcons name="content-save" color="white" size={20} />
                 <TextRegular textStyle={styles.text}>Save</TextRegular>
              </View>
            </Pressable>

          </View>
        </View>
      </ScrollView>
    )}
  </Formik>
)
```

---

## TAREA 4: Ajustar Estilos y Analíticas (Flexbox)

**Objetivo:** Mostrar objetos dinámicos o ajustar la maquetación.

### 1. Renderizado Condicional del Componente
Si una sección depende de datos que tardan en cargar (analíticas), asegúrate de que el estado no sea `null` antes de pintarlo.
```javascript
// En el JSX principal, donde vas a llamar a la cabecera/analíticas:
{ analytics !== null && renderAnalytics() }
```

### 2. Imprimir valores
Si es un número y quieres decimales fijos, utiliza la función `.toFixed(2)` en el JS: `{analytics.invoicedToday.toFixed(2)}€`.

### 3. Estilos Flexbox (`StyleSheet.create`)
Para que dos contenedores se coloquen en la misma fila con espacio entre ellos (típico en métricas o botones en horizontal):
```javascript
analyticsRow: {
  flexDirection: 'row', // Los pone en la misma fila (horizontal)
  justifyContent: 'space-around' // space-between o space-around distribuyen el espacio
}
```

---
¡Guarda este documento abierto! Con esto tendrás plantillas literales de cómo se configura Formik con API, cómo forzar recargas y cómo montar listados. ¡No fallarás!
