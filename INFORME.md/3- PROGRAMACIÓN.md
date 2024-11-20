# Codigo

#include <stdio.h>
#include <freertos/FreeRTOS.h>
#include <freertos/task.h>
#include <esp_wifi.h>
#include <nvs_flash.h>
#include <mqtt_client.h>
#include <driver/gpio.h>
#include <driver/adc.h>

// Pines
#define LASER1_PIN GPIO_NUM_13
#define LASER2_PIN GPIO_NUM_14
#define FOTO1_PIN ADC_CHANNEL_6  // GPIO 34
#define FOTO2_PIN ADC_CHANNEL_7  // GPIO 35
#define STEP1_PIN GPIO_NUM_18
#define DIR1_PIN GPIO_NUM_5
#define STEP2_PIN GPIO_NUM_17
#define DIR2_PIN GPIO_NUM_16
#define EMERGENCIA_PIN GPIO_NUM_25

// Configuración de MQTT
#define BROKER_URI "mqtt://broker.hivemq.com"
#define TOPIC_ALTO "cinta/alto"
#define TOPIC_BAJO "cinta/bajo"

// Variables globales
esp_mqtt_client_handle_t mqtt_client;
volatile bool sistema_activo = true;

// Funciones
void leerSensoresTask(void *pvParameters);
void controlMotoresTask(void *pvParameters);
void mqttTask(void *pvParameters);
void iniciarWiFi();
void iniciarMQTT();
void enviarMensaje(const char *topic, const char *mensaje);

void app_main() {
    // Inicializar NVS para WiFi
    esp_err_t ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);

    iniciarWiFi();
    iniciarMQTT();

    // Configuración de pines
    gpio_set_direction(LASER1_PIN, GPIO_MODE_OUTPUT);
    gpio_set_direction(LASER2_PIN, GPIO_MODE_OUTPUT);
    gpio_set_direction(STEP1_PIN, GPIO_MODE_OUTPUT);
    gpio_set_direction(DIR1_PIN, GPIO_MODE_OUTPUT);
    gpio_set_direction(STEP2_PIN, GPIO_MODE_OUTPUT);
    gpio_set_direction(DIR2_PIN, GPIO_MODE_OUTPUT);
    gpio_set_direction(EMERGENCIA_PIN, GPIO_MODE_INPUT);

    // Encender láseres
    gpio_set_level(LASER1_PIN, 1);
    gpio_set_level(LASER2_PIN, 1);

    // Crear tareas
    xTaskCreate(leerSensoresTask, "LeerSensoresTask", 2048, NULL, 5, NULL);
    xTaskCreate(controlMotoresTask, "ControlMotoresTask", 2048, NULL, 5, NULL);
    xTaskCreate(mqttTask, "MQTTTask", 4096, NULL, 5, NULL);
}

void leerSensoresTask(void *pvParameters) {
    const int umbral = 800;  // Umbral para las fotoresistencias
    while (true) {
        if (gpio_get_level(EMERGENCIA_PIN) == 0) {
            sistema_activo = false;
            printf("Sistema detenido por emergencia\n");
            vTaskDelay(pdMS_TO_TICKS(1000));
            continue;
        }
        if (sistema_activo) {
            int lecturaFoto1 = adc1_get_raw(FOTO1_PIN);
            int lecturaFoto2 = adc1_get_raw(FOTO2_PIN);

            if (lecturaFoto1 < umbral) {
                enviarMensaje(TOPIC_BAJO, "1");
                printf("Caja baja detectada\n");
            }
            if (lecturaFoto2 < umbral) {
                enviarMensaje(TOPIC_ALTO, "1");
                printf("Caja alta detectada\n");
            }
        }
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}

void controlMotoresTask(void *pvParameters) {
    while (true) {
        if (sistema_activo) {
            // Motor 1
            gpio_set_level(DIR1_PIN, 1);
            gpio_set_level(STEP1_PIN, 1);
            vTaskDelay(pdMS_TO_TICKS(1));
            gpio_set_level(STEP1_PIN, 0);
            vTaskDelay(pdMS_TO_TICKS(1));

            // Motor 2
            gpio_set_level(DIR2_PIN, 1);
            gpio_set_level(STEP2_PIN, 1);
            vTaskDelay(pdMS_TO_TICKS(1));
            gpio_set_level(STEP2_PIN, 0);
            vTaskDelay(pdMS_TO_TICKS(1));
        } else {
            gpio_set_level(STEP1_PIN, 0);
            gpio_set_level(STEP2_PIN, 0);
        }
    }
}

void mqttTask(void *pvParameters) {
    while (true) {
        esp_mqtt_client_start(mqtt_client);
        vTaskDelay(pdMS_TO_TICKS(10000));  // Reintento cada 10 segundos
    }
}

void enviarMensaje(const char *topic, const char *mensaje) {
    if (mqtt_client) {
        esp_mqtt_client_publish(mqtt_client, topic, mensaje, 0, 1, 0);
    }
}

void iniciarWiFi() {
    esp_netif_init();
    esp_event_loop_create_default();
    esp_netif_create_default_wifi_sta();

    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    esp_wifi_init(&cfg);

    wifi_config_t wifi_config = {
        .sta = {
            .ssid = "Tu_SSID",
            .password = "Tu_PASSWORD"
        },
    };
    esp_wifi_set_mode(WIFI_MODE_STA);
    esp_wifi_set_config(ESP_IF_WIFI_STA, &wifi_config);
    esp_wifi_start();

    printf("Conectando a WiFi...\n");
    while (esp_wifi_connect() != ESP_OK) {
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
    printf("WiFi conectado\n");
}

void iniciarMQTT() {
    // Configuración MQTT (sin los campos 'uri' ni 'event_handle')
    esp_mqtt_client_config_t mqtt_cfg = {
        .broker.address.uri = BROKER_URI,  // Cambiado de 'uri' a 'broker.address.uri'
    };

    // Inicializamos el cliente MQTT
    mqtt_client = esp_mqtt_client_init(&mqtt_cfg);
}
