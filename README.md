# GrowiiClients

El objetivo de este repositorio es almacenar los clientes para los diversos brokers que se puedan necesitar de tal manera que exista una manera unificada de conexión para todos los que trabajamos en este proyecto, facilitando el mantenimiento, las mejoras y permitiendo la compatibilidad de estrategias de inversión de manera sencilla.

Actualmente, los brokers incluidos en este proyecto son:

**1. Interactive Brokers**

Este proyecto se ha diseñado para uso del Departamento de Matemática Aplicada de la Universidad Politécnica de Madrid. 

1.Interactive Brokers
------
Interactive Brokers permite, mediante su API, el uso de varios lenguajes de programación para interactuar con él, como son Java, C# y Python. La información para todos ellos es concisa y facil de encontrar [aquí](https://interactivebrokers.github.io/tws-api/index.html)

En este caso, se ha optado por Python como lenguaje de desarrollo del cliente para facilitar la conexión. A continuación, una pequeña descripción del funcionamiento.

### Descripción

Interactive Brokers mantiene dos aplicaciones distintas diseñadas para el trading algorítmico. Una de ellas es TWS (Trader WorkStation) y la otra es IBGateway. A priori, la conexión con ambas es similar. Ambas se pueden descargar desde la página de IB, y en ambas está soportado tanto el trading real como el paper trading. No se permite la ejecución de ambas usando el mismo cliente (aun en equipos distintos). No se permite operar en real y en paper trading con la misma cuenta. Y no se permite usar el mismo cliente desde IPs distintas.

Las mayores diferencias entre ambas son su interfaz gráfica, y la manera de interactuar con ellas. En el caso de la interfaz gráfica de TWS es una interfaz completa, con mucha información y con gran capacidad de personalización. Es bastante pesada pero permite seguir con claridad cual es la evolución de los algoritmos. Por contra, IBGateway tiene una interfaz muy ligera que sólo permite comprobar el estado de la conectividad. Además, para interactuar con ella el protocolo de comunicación es mucho más estricto, por lo que su uso se recomienda sólo cuando se posea ya cierta destreza en el trading algorítmico. Por último, es imprescindible tener en ejecución una de las dos para poder interactuar con IB. Esto es una decisión de diseño de IB que justifican por motivos de seguridad.


A la hora de interactuar con TWS, tienen un protocolo de comunicación muy claro, y al comprobar su documetación se verá que el sistema se basa en comunicaciones unidireccionales. Es decir: Existe un grupo de funciones, cuyo retorno es 'void' que permiten el enviar mensajes desde nuestro código, a los servidores de IB. Dichas funciones tendrán de entrada todos los argumentos que el servidor pudiese necesitar para cumplir con el propósito que queramos. A su vez, existe un grupo de funciones que no tienen argumentos de entrada que lo que permiten es la recepción de la respuesta del servidor. Este es un sistema bastante robusto, con la salvedad de que queda en nuestra mano el como gestionar los tiempo de espera entre ambas funciones.

En el caso de TWS, el cliente se ha diseñado de tal manera que se simplifique la interacción con él, quedando la gestión de retardos dentro del cliente, simplificando su visualización y su gestión. Para ello, dentro del cliente se han incorporado una serie de métodos, que bajo la clásica estructura de 'set', 'get' y 'check' permita una interacción completa con IB. A lo largo de este documento, se irán detallando los métodos y objetos que incluye el cliente.

### Métodos públicos

#### Constructor
El constructor es un método que se ejecuta siempre al crear un cliente. Los argumentos de entrada son la dirección IP ('127.0.0.1' si se desea usar en local), el puerto habilitado para la conexión (debe coincidir con el hablitado en TWS) y un ID, que permitirá identificar todas las interacciones con IB. Es imprescindible que cada estrategia use única y exclusivamente un ID de manera constante, permitiendo así evaluar su rendimiento. IB permite hasta un máximo de 32 conexiones simultaneas, permitiendonos por tanto hasta 32 estrategias de manera simultanea (siempre y cuando seamos consistentes con el uso del cliente).

A lo largo de esta función, lo que se hace es guardar tanto la IP, como el puerto y el ID como miembros de la clase, así como establecer la conexión con el servidor. No se espera ningún argumento de respuesta, pero si una serie de mensajes confirmando que la conexión ha sido realizada correctamente.
```python
    def __init__(self, addr, port, client_id):
        """
        CONSTRUCTOR
        
        Parameters
        ----------
        addr : str
            IP where TWS or IBGateway is running.
        port : int
            Firewall port where is allowed the connection
        client_id : int
            Number beetween 1 and 32. This allow several clients connected to
            a TWS session. This number should be fixed for the admin in the server
            For test, 1 is enought.
            
        """
        self.client_id = client_id
        self.address = addr
        self.port = port
        
        EClient. __init__(self, self)
        EWrapper.__init__(self)

        # Connect to TWS
        self.connect(addr, port, client_id)
        
        # Launch the client thread
        thread = Thread(target=self.run)
        thread.start()
```

#### check_conection
El objetivo de esta función es comprobar si la conexión con el servidor está establecida. Y si no, intenta reconectarse cada 5 segundos hasta que lo logra.
Es necesario que siempre que haya un tiempo de espera considerable entre interacciones con el servidor, este método sea el primero en ejecutarse. Si no, es posible que se generen errores durante la ejecución.
En la versión actual (12/12/20) si no está establecida la conexión, intenta reconectarse cada 5 segundos. Es posible que esto pueda generar problemas en un futuro. De ser así, se actualizará.
```python
    def check_conection(self):
        """
        This function checks if the connection with the TWS or IB Gateway 
        application and if it is not available, it reconnects.
        
        """
        retFlag = self.isConnected()
        
        # i = 0 # Falta añadir un máximo número de intentos
        while retFlag != True:
            print('{}    Conexión con el servidor no disponible'.format(datetime.now()))
            self.connect('127.0.0.1', 7497, 1)
            
            # Launch the client thread
            thread = Thread(target=self.run)
            thread.start()
            
            time.sleep(5)
            retFlag = self.isConnected()
        return
```

#### getHistoricalData
Esta función lo que permite es la descarga de datos históricos a través de IB. Una de las limitaciones que da IB es que no es un servicio de provisión de datos financieros, por lo que 

```python
def getHistoricalData(self, delay, contract, endDateTime, durationStr, barSizeSetting, whatToShow):
        """
        This function allows you to request a historical data from the server. 
        The greater the number of data, the greater the margin delay that must
        be provided to the server. In case the time does not reach, an empty 
        list is returned

        Parameters
        ----------
        contract : Contract
            TWS Contract object
        endDateTime : str
            End date of the request
        durationStr : TYPE
            Lenght of the requested data
        barSizeSetting : TYPE
            Step beetwen data
        whatToShow : TYPE
            Ask, bid o midpoint

        Returns
        -------
        hist_date : float list
            List with requested date
        hist_open : float list
            List with requested open values
        hist_high : float list
            List with requested high values
        hist_low : float list
            List with requested low values
        hist_close : float list
            List with requested close values
        hist_volume : float list
            List with requested volume of operations

        """
        self.historical_data = []
        self.reqHistoricalData(self.client_id, contract, endDateTime, durationStr, barSizeSetting, whatToShow, False, 1, False, [])
        time.sleep(delay)
        
        hist_date = [instant.date for instant in self.historical_data]
        hist_open = [instant.open for instant in self.historical_data]
        hist_high = [instant.high for instant in self.historical_data]
        hist_low = [instant.low for instant in self.historical_data]
        hist_close = [instant.close for instant in self.historical_data]
        hist_volume = [instant.volume for instant in self.historical_data]
        
        return (hist_date, hist_open , hist_high , hist_low , hist_close, hist_volume)
```
