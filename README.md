# Example นี้มีชื่อว่า BLE_ANCS
ซึ่ง Exampleนี้ เป็นการทำให้ esp32 ปล่อยสัญญานออกมาแบบBlutooth เพื่อดักจับสัญญาณเช่น การโทรเบอร์โทรศัพท์
# ขั้นตอนที่1-3 ดังรูปต่อไปนี้
# ![รูปภาพ (9)-1 2](https://github.com/user-attachments/assets/fa4bdca4-674b-4835-82b4-a8112afd5758)
## ขั้นตอนที่4 ทำการbuild 
# ขั้นตอนที่5 เชื่อมต่อblutooth
-โดยนำโทรศัพท์มือถือเชื่อมต่อblutooth จะได้ดังรูป
#  ![IMG_7248](https://github.com/user-attachments/assets/9ef8cbe3-3ee9-4afe-b7e4-2b6d8f021be5)
# ขั้นตอนที่6 ทดลองการดักจับสัญญาณ
#  <img width="795" alt="ภาพถ่ายหน้าจอ 2567-11-01 เวลา 22 44 54" src="https://github.com/user-attachments/assets/81ff8a59-7dd0-42ab-a07b-ae1fad2a9ab6">

## ปรับปรุงแก้ไข้ในการใช้งานได้ดังนี้
หากมีการใช้ malloc ในการจัดการหน่วยความจำแบบไดนามิก ควรคืนค่าหน่วยความจำโดยใช้ free ทุกครั้งหลังใช้งานเสร็จ
#  <img width="798" alt="ภาพถ่ายหน้าจอ 2567-11-01 เวลา 23 06 24" src="https://github.com/user-attachments/assets/9cb8dea2-9a6a-496f-8ddd-0f2b663a5858">
```
#include <stdlib.h>
#include <string.h>
#include <inttypes.h>
#include "esp_log.h"
#include "ble_ancs.h"

#define BLE_ANCS_TAG  "BLE_ANCS"

char *EventID_to_String(uint8_t EventID) {
    char *str = malloc(20);
    if (!str) return NULL;
    switch (EventID) {
        case EventIDNotificationAdded:
            strcpy(str, "New message");
            break;
        case EventIDNotificationModified:
            strcpy(str, "Modified message");
            break;
        case EventIDNotificationRemoved:
            strcpy(str, "Removed message");
            break;
        default:
            strcpy(str, "unknown EventID");
            break;
    }
    return str;
}

char *CategoryID_to_String(uint8_t CategoryID) {
    char *Cidstr = malloc(25);
    if (!Cidstr) return NULL;
    switch (CategoryID) {
        case CategoryIDOther:
            strcpy(Cidstr, "Other");
            break;
        case CategoryIDIncomingCall:
            strcpy(Cidstr, "Incoming Call");
            break;
        case CategoryIDMissedCall:
            strcpy(Cidstr, "Missed Call");
            break;
        case CategoryIDVoicemail:
            strcpy(Cidstr, "Voicemail");
            break;
        case CategoryIDSocial:
            strcpy(Cidstr, "Social");
            break;
        case CategoryIDSchedule:
            strcpy(Cidstr, "Schedule");
            break;
        case CategoryIDEmail:
            strcpy(Cidstr, "Email");
            break;
        case CategoryIDNews:
            strcpy(Cidstr, "News");
            break;
        case CategoryIDHealthAndFitness:
            strcpy(Cidstr, "Health & Fitness");
            break;
        case CategoryIDBusinessAndFinance:
            strcpy(Cidstr, "Business & Finance");
            break;
        case CategoryIDLocation:
            strcpy(Cidstr, "Location");
            break;
        case CategoryIDEntertainment:
            strcpy(Cidstr, "Entertainment");
            break;
        default:
            strcpy(Cidstr, "Unknown CategoryID");
            break;
    }
    return Cidstr;
}

char *Errcode_to_String(uint16_t status) {
    char *Errstr = malloc(25);
    if (!Errstr) return NULL;
    switch (status) {
        case Unknown_command:
            strcpy(Errstr, "Unknown command");
            break;
        case Invalid_command:
            strcpy(Errstr, "Invalid command");
            break;
        case Invalid_parameter:
            strcpy(Errstr, "Invalid parameter");
            break;
        case Action_failed:
            strcpy(Errstr, "Action failed");
            break;
        default:
            strcpy(Errstr, "Unknown error");
            break;
    }
    return Errstr;
}

void esp_receive_apple_notification_source(uint8_t *message, uint16_t message_len) {
    if (!message || message_len < 5) {
        return;
    }

    uint8_t EventID = message[0];
    char *EventIDS = EventID_to_String(EventID);
    if (!EventIDS) return;

    uint8_t EventFlags = message[1];
    uint8_t CategoryID = message[2];
    char *Cidstr = CategoryID_to_String(CategoryID);
    if (!Cidstr) {
        free(EventIDS);
        return;
    }

    uint8_t CategoryCount = message[3];
    uint32_t NotificationUID = message[4] | (message[5] << 8) | (message[6] << 16) | (message[7] << 24);

    ESP_LOGI(BLE_ANCS_TAG, "EventID:%s EventFlags:0x%x CategoryID:%s CategoryCount:%d NotificationUID:%" PRIu32,
             EventIDS, EventFlags, Cidstr, CategoryCount, NotificationUID);

    free(EventIDS);
    free(Cidstr);
}

void esp_receive_apple_data_source(uint8_t *message, uint16_t message_len) {
    if (!message || message_len == 0) {
        return;
    }

    uint8_t Command_id = message[0];
    switch (Command_id) {
        case CommandIDGetNotificationAttributes: {
            uint32_t NotificationUID = (message[1]) | (message[2] << 8) | (message[3] << 16) | (message[4] << 24);
            uint32_t remian_attr_len = message_len - 5;
            uint8_t *attrs = &message[5];
            ESP_LOGI(BLE_ANCS_TAG, "recevice Notification Attributes response Command_id %d NotificationUID %" PRIu32, Command_id, NotificationUID);

            while (remian_attr_len > 0) {
                uint8_t AttributeID = attrs[0];
                uint16_t len = attrs[1] | (attrs[2] << 8);
                if (len > (remian_attr_len - 3)) {
                    ESP_LOGE(BLE_ANCS_TAG, "data error");
                    break;
                }
                esp_log_buffer_char("AttributeID Data", &attrs[3], len);
                attrs += (1 + 2 + len);
                remian_attr_len -= (1 + 2 + len);
            }
            break;
        }
        case CommandIDGetAppAttributes:
            ESP_LOGI(BLE_ANCS_TAG, "recevice APP Attributes response");
            break;
        case CommandIDPerformNotificationAction:
            ESP_LOGI(BLE_ANCS_TAG, "recevice Perform Notification Action");
            break;
        default:
            ESP_LOGI(BLE_ANCS_TAG, "Unknown Command ID");
            break;
    }
}

```
