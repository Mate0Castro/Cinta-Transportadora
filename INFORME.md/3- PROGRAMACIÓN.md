# Codigo

#include <stdio.h>                      // Biblioteca estándar para entrada/salida
#include <stdint.h>                     // Tipos de datos enteros específicos
#include <stddef.h>                     // Definiciones básicas como NULL
#include <string.h>                     // Funciones para manejar cadenas
#include "esp_wifi.h"                   // Biblioteca para configurar WiFi
#include "esp_system.h"                 // Funciones básicas del sistema
#include "nvs_flash.h"                  // Biblioteca para memoria no volátil
#include "esp_event.h"                  // Gestión de eventos del sistema
#include "esp_netif.h"                  // Interfaz de red
#include "freertos/FreeRTOS.h"          // Biblioteca principal de FreeRTOS
#include "freertos/task.h"              // Gestión de tareas en FreeRTOS
#include "freertos/semphr.h"            // Semáforos en FreeRTOS
#include "freertos/queue.h"             // Colas en FreeRTOS
#include "esp_log.h"                    // Biblioteca para registro de eventos
#include "mqtt_client.h"                // Biblioteca para cliente MQTT
#include "driver/gpio.h"                // Control de pines GPIO
#include "driver/adc.h"                 // Conversión analógica a digital (ADC)
#include "esp_adc_cal.h"                // Calibración del ADC
#include "wifi_credentials.h"          // Archivo externo con credenciales WiFi (definido por el usuario)

// Definición de una etiqueta de registro para depuración
static const char *TAG = "CONVEYOR";

// Pines utilizados en el sistema
#define PHOTO_SENSOR_BOT    ADC1_CHANNEL_6  // GPIO34 - Fotoresistencia inferior
#define PHOTO_SENSOR_TOP    ADC1_CHANNEL_7  // GPIO35 - Fotoresistencia superior
#define LASER1_PIN          GPIO_NUM_14     // Láser 1
#define LASER2_PIN          GPIO_NUM_13     // Láser 2
#define MOTOR1_STEP_PIN     GPIO_NUM_18     // Motor 1 STEP
#define MOTOR1_DIR_PIN      GPIO_NUM_5      // Motor 1 DIR
#define MOTOR2_STEP_PIN     GPIO_NUM_17     // Motor 2 STEP
#define MOTOR2_DIR_PIN      GPIO_NUM_16     // Motor 2 DIR
#define EMERGENCY_STOP      GPIO_NUM_25     // Pulsador de emergencia

// Variables globales
static bool emergency_stopped = false;  // Indica si se activó el paro de emergencia
static int object_height = 0;          // Altura del objeto detectado
static esp_mqtt_client_handle_t mqtt_client = NULL;  // Cliente MQTT
static TaskHandle_t motor_task_handle = NULL;        // Handle para la tarea del motor
static TaskHandle_t sensor_task_handle = NULL;       // Handle para la tarea del sensor

// Configuración WiFi y MQTT (se definen en wifi_credentials.h)
#define WIFI_SSID       CONFIG_WIFI_SSID
#define WIFI_PASSWORD   CONFIG_WIFI_PASSWORD
#define MQTT_BROKER_URL CONFIG_MQTT_BROKER_URL
#define MQTT_USERNAME   CONFIG_MQTT_USERNAME
#define MQTT_PASSWORD   CONFIG_MQTT_PASSWORD
#define MQTT_TOPIC      "cinta/estado"

// Función para inicializar el WiFi
static void wifi_init(void)
{
    ESP_ERROR_CHECK(esp_netif_init());  // Inicializa la interfaz de red
    ESP_ERROR_CHECK(esp_event_loop_create_default());  // Crea el bucle de eventos predeterminado
    esp_netif_create_default_wifi_sta();  // Crea la interfaz de estación WiFi

    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();  // Configuración por defecto del WiFi
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));  // Inicializa el WiFi con la configuración

    // Configuración de WiFi
    wifi_config_t wifi_config = {
        .sta = {
            .marisani = WIFI_SSID,          // SSID de la red WiFi
            .fatiga2409 = WIFI_PASSWORD,  // Contraseña de la red WiFi
        },
    };

    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));  // Configura el modo de estación
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_STA, &wifi_config));  // Aplica la configuración
    ESP_ERROR_CHECK(esp_wifi_start());  // Inicia la conexión WiFi

    ESP_LOGI(TAG, "Conectando a WiFi...");  // Mensaje de registro
}

    // Esperar conexión WiFi
    EventBits_t bits = xEventGroupWaitBits(wifi_event_group,
            WIFI_CONNECTED_BIT | WIFI_FAIL_BIT,  // Eventos esperados: conexión o falla
            pdFALSE,                            // No limpiar los bits después de esperar
            pdFALSE,                            // No esperar ambos eventos simultáneamente
            portMAX_DELAY);                     // Esperar indefinidamente

    if (bits & WIFI_CONNECTED_BIT) {           // Si se conecta exitosamente
        ESP_LOGI(TAG, "Conectado a WiFi");     // Mensaje de éxito
    } else {                                   // Si ocurre un error
        ESP_LOGE(TAG, "Error conectando a WiFi");  // Mensaje de error
    }
}

// Función para encender o apagar los láseres
static void control_lasers(bool estado) {
    gpio_set_level(LASER1_PIN, estado);  // Establece el estado del Láser 1
    gpio_set_level(LASER2_PIN, estado);  // Establece el estado del Láser 2
}

// Tarea que controla los motores paso a paso
static void motor_task(void *pvParameters) {
    const uint8_t STEP_DELAY = 2;       // Tiempo de retardo entre pasos (en microsegundos)
    bool direction = true;             // Dirección inicial de los motores
    
    gpio_set_level(MOTOR1_DIR_PIN, direction);  // Establece la dirección del Motor 1
    gpio_set_level(MOTOR2_DIR_PIN, direction);  // Establece la dirección del Motor 2
    
    while (1) {                        // Bucle infinito
        if (!emergency_stopped) {      // Solo mueve los motores si no hay paro de emergencia
            gpio_set_level(MOTOR1_STEP_PIN, 1);  // Genera un pulso de paso para el Motor 1
            gpio_set_level(MOTOR2_STEP_PIN, 1);  // Genera un pulso de paso para el Motor 2
            ets_delay_us(STEP_DELAY);           // Retardo entre niveles altos y bajos
            gpio_set_level(MOTOR1_STEP_PIN, 0);  // Finaliza el pulso para el Motor 1
            gpio_set_level(MOTOR2_STEP_PIN, 0);  // Finaliza el pulso para el Motor 2
            ets_delay_us(STEP_DELAY);           // Retardo entre pulsos
        }
        vTaskDelay(pdMS_TO_TICKS(1));  // Pausa la tarea brevemente para liberar CPU
    }
}

// Función para enviar mensajes MQTT con el estado del sistema
static void publish_status(const char* status) {
    if (mqtt_client) {  // Verifica que el cliente MQTT esté inicializado
        int msg_id = esp_mqtt_client_publish(mqtt_client, MQTT_TOPIC, status, 0, 1, 0);  
        // Publica el estado en el tópico MQTT con QoS 1
        ESP_LOGI(TAG, "Mensaje enviado, ID: %d - Estado: %s", msg_id, status);  // Registro del mensaje enviado
    }
}

// Tarea para leer los sensores y determinar la altura del objeto
static void check_sensors(void *pvParameters) {
    esp_adc_cal_characteristics_t adc1_chars;  // Características del ADC para calibración
    esp_adc_cal_characterize(ADC_UNIT_1, ADC_ATTEN_DB_11, ADC_WIDTH_BIT_12, 0, &adc1_chars);
    // Configura el ADC con la resolución y atenuación correctas
    
    while (1) {  // Bucle infinito
        if (!emergency_stopped) {  // Solo lee los sensores si no hay paro de emergencia
            uint32_t bottom_value = adc1_get_raw(PHOTO_SENSOR_BOT);  // Lee el valor del sensor inferior
            uint32_t top_value = adc1_get_raw(PHOTO_SENSOR_TOP);     // Lee el valor del sensor superior
            
            const int threshold = 2000;  // Umbral para determinar si el láser está bloqueado
            
            ESP_LOGD(TAG, "Sensores - Superior: %ld, Inferior: %ld", top_value, bottom_value);
            // Registra los valores de los sensores para depuración
            
            // Determina la altura del objeto basado en los sensores
            if (top_value < threshold && bottom_value < threshold) {
                if (object_height != 2) {
                    object_height = 2;  // Objeto alto detectado
                    publish_status("alto");  // Publica el estado "alto" en MQTT
                }
            } else if (bottom_value < threshold) {
                if (object_height != 1) {
                    object_height = 1;  // Objeto bajo detectado
                    publish_status("bajo");  // Publica el estado "bajo" en MQTT
                }
            } else if (object_height != 0) {
                object_height = 0;  // Sin objeto detectado
                publish_status("sin_objeto");  // Publica el estado "sin_objeto" en MQTT
            }
        }
        vTaskDelay(pdMS_TO_TICKS(100));  // Pausa de 100 ms antes de la próxima lectura
    }
}

// Interrupción para el paro de emergencia
static void IRAM_ATTR emergency_stop_isr(void* arg) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;  // Bandera para cambio de prioridad
    emergency_stopped = !emergency_stopped;  // Cambia el estado de emergencia

    if (emergency_stopped) {  // Si se activa la emergencia
        control_lasers(false);       // Apaga los láseres
        publish_status("emergency_stop");  // Publica el estado "emergency_stop" en MQTT
    } else {  // Si se desactiva la emergencia
        control_lasers(true);        // Enciende los láseres
        publish_status("running");   // Publica el estado "running" en MQTT
    }
    
    if (xHigherPriorityTaskWoken) {  // Si hay una tarea con mayor prioridad
        portYIELD_FROM_ISR();        // Cede el control de CPU a esa tarea
    }
}

// Manejo de eventos MQTT
static void mqtt_event_handler(void *handler_args, esp_event_base_t base, int32_t event_id, void *event_data) {
    esp_mqtt_event_handle_t event = event_data;  // Obtiene los datos del evento
    switch (event->event_id) {
        case MQTT_EVENT_CONNECTED:  // Evento de conexión MQTT
            ESP_LOGI(TAG, "MQTT Conectado");
            publish_status("connected");  // Publica el estado "connected"
            break;
        case MQTT_EVENT_DISCONNECTED:  // Evento de desconexión MQTT
            ESP_LOGI(TAG, "MQTT Desconectado");
            break;
        default:
            break;
    }
}

// Configuración inicial de GPIO
static void init_gpio(void) {
    gpio_config_t io_conf_output = {
        .intr_type = GPIO_INTR_DISABLE,  // Sin interrupciones para pines de salida
        .mode = GPIO_MODE_OUTPUT,        // Configura como salida
        .pin_bit_mask = (1ULL<<MOTOR1_STEP_PIN) | (1ULL<<MOTOR1_DIR_PIN) |
                        (1ULL<<MOTOR2_STEP_PIN) | (1ULL<<MOTOR2_DIR_PIN) |
                        (1ULL<<LASER1_PIN) | (1ULL<<LASER2_PIN),
        .pull_down_en = 0,               // Sin resistencia pull-down
        .pull_up_en = 0                  // Sin resistencia pull-up
    };
    gpio_config(&io_conf_output);  // Aplica configuración

    gpio_config_t io_conf_input = {
        .intr_type = GPIO_INTR_ANYEDGE,  // Interrupción en cualquier borde
        .mode = GPIO_MODE_INPUT,         // Configura como entrada
        .pin_bit_mask = (1ULL<<EMERGENCY_STOP),  // Configura el pin de emergencia
        .pull_up_en = 1                 // Activa resistencia pull-up
    };
    gpio_config(&io_conf_input);  // Aplica configuración
    
    adc1_config_width(ADC_WIDTH_BIT_12);          // Configura el ADC a 12 bits
    adc1_config_channel_atten(PHOTO_SENSOR_BOT, ADC_ATTEN_DB_11);  // Atenuación para el sensor inferior
    adc1_config_channel_atten(PHOTO_SENSOR_TOP, ADC_ATTEN_DB_11);  // Atenuación para el sensor superior
    
    gpio_install_isr_service(0);                  // Inicializa el servicio de interrupciones
    gpio_isr_handler_add(EMERGENCY_STOP, emergency_stop_isr, NULL);  // Añade interrupción al botón de emergencia
    
    control_lasers(true);  // Enciende los láseres por defecto
}
static void init_mqtt(void) {
    esp_mqtt_client_config_t mqtt_cfg = {
        .broker.address.uri = MQTT_URI,  // URI del broker MQTT
    };
    
    mqtt_client = esp_mqtt_client_init(&mqtt_cfg);  // Inicializa el cliente MQTT
    esp_mqtt_client_register_event(mqtt_client, ESP_EVENT_ANY_ID, mqtt_event_handler, NULL);
    // Registra el manejador de eventos MQTT
    esp_mqtt_client_start(mqtt_client);  // Inicia el cliente MQTT
}
void app_main(void) {
    // Inicializa NVS (almacenamiento no volátil), necesario para WiFi y MQTT
    esp_err_t ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());  // Borra la memoria NVS si es necesario
        ESP_ERROR_CHECK(nvs_flash_init());
    }

    // Inicializa la conexión WiFi
    wifi_init_sta();  

    // Inicializa los GPIOs
    init_gpio();

    // Inicializa MQTT
    init_mqtt();

    // Crea la tarea para controlar los motores paso a paso
    xTaskCreate(motor_task, "motor_task", 2048, NULL, 5, NULL);

    // Crea la tarea para leer los sensores de altura
    xTaskCreate(check_sensors, "check_sensors", 2048, NULL, 5, NULL);

    // Publica un mensaje indicando que el sistema está en funcionamiento
    publish_status("running");
}
