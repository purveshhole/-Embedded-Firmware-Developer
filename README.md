
#include <stdio.h>
#include <string.h>
#include <sys/stat.h>
#include "esp_log.h"
#include "esp_err.h"
#include "esp_vfs.h"
#include "esp_vfs_fat.h"
#include "esp_spiffs.h"
#include "esp_http_client.h"
#include "esp_system.h"
#include "nvs_flash.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#define FILE_PATH "/spiffs/downloaded_file.bin"
#define BUFFER_SIZE 4096  // Optimized buffer size for performance
#define DOWNLOAD_URL "https://example.com/sample.bin"  // Replace with your file URL

static const char *TAG = "HTTPS_DOWNLOAD";

// SPIFFS initialization
void init_spiffs() {
    ESP_LOGI(TAG, "Initializing SPIFFS");
    esp_vfs_spiffs_conf_t conf = {
        .base_path = "/spiffs",
        .partition_label = NULL,
        .max_files = 5,
        .format_if_mount_failed = true
    };
    esp_err_t ret = esp_vfs_spiffs_register(&conf);
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "Failed to mount SPIFFS (%s)", esp_err_to_name(ret));
    } else {
        ESP_LOGI(TAG, "SPIFFS Mounted Successfully");
    }
}

// HTTPS Download Event Handler
esp_err_t _http_event_handler(esp_http_client_event_t *evt) {
    return ESP_OK;
}

void download_file_task(void *pvParameters) {
    FILE *file = fopen(FILE_PATH, "wb");
    if (!file) {
        ESP_LOGE(TAG, "Failed to open file for writing");
        vTaskDelete(NULL);
        return;
    }
    
    esp_http_client_config_t config = {
        .url = DOWNLOAD_URL,
        .event_handler = _http_event_handler,
    };
    esp_http_client_handle_t client = esp_http_client_init(&config);
    
    esp_err_t err = esp_http_client_open(client, 0);
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "Failed to open HTTP connection: %s", esp_err_to_name(err));
        fclose(file);
        vTaskDelete(NULL);
        return;
    }
    
    int content_length = esp_http_client_fetch_headers(client);
    ESP_LOGI(TAG, "File size: %d bytes", content_length);
    
    uint8_t buffer[BUFFER_SIZE];
    int total_bytes = 0;
    int read_bytes;
    int64_t start_time = esp_timer_get_time();
    
    while ((read_bytes = esp_http_client_read(client, (char *)buffer, BUFFER_SIZE)) > 0) {
        fwrite(buffer, 1, read_bytes, file);
        total_bytes += read_bytes;
    }
    
    int64_t end_time = esp_timer_get_time();
    fclose(file);
    esp_http_client_close(client);
    esp_http_client_cleanup(client);
    
    double elapsed_time = (end_time - start_time) / 1000000.0;
    double speed_kbps = (total_bytes / elapsed_time) / 1024.0;
    ESP_LOGI(TAG, "Download speed: %.2f KBps", speed_kbps);
    
    if (speed_kbps < 400.0) {
        ESP_LOGW(TAG, "Download speed is below requirement!");
    }
    
    vTaskDelete(NULL);
}

void app_main() {
    ESP_ERROR_CHECK(nvs_flash_init());
    init_spiffs();
    xTaskCreate(&download_file_task, "download_file_task", 8192, NULL, 5, NULL);
}
