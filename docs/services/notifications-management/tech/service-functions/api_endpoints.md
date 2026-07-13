# وظائف الخدمة: واجهات برمجية (APIs)

## تعريفات الإشعارات (منصة)
- POST /api/admin/notification-definitions
- GET /api/admin/notification-definitions
- GET /api/admin/notification-definitions/{key}
- PUT /api/admin/notification-definitions/{key}
- DELETE /api/admin/notification-definitions/{key}

## قوالب الإشعارات (منصة/جهة)
- POST /api/{scope}/notification-templates
- GET /api/{scope}/notification-templates
- PUT /api/{scope}/notification-templates/{id}
- DELETE /api/{scope}/notification-templates/{id}

حيث {scope} ∈ {admin, tenant}

## تفضيلات الإشعارات (مستخدم)
- GET /api/me/notification-preferences
- PUT /api/me/notification-preferences/{definition_key}/{channel}

## تشغيل إشعار
- POST /api/notifications/trigger
  - body: { definition_key, recipient, data, channels? }

## إشعارات التطبيق (Firebase Messaging)
- POST /api/me/notification-tokens
  - body: { token, platform, app_version? }
- DELETE /api/me/notification-tokens/{token}
- GET /api/me/notifications
- GET /api/me/notifications/unread-count
- PUT /api/me/notifications/{id}/read
- PUT /api/me/notifications/read-all

