# RindexWorkflow

## Назначение
RindexWorkflow — это связка узлов ComfyUI для SDXL, ориентированная на детальную отрисовку персонажей в аниме-стилистике с постобработкой рук, лица и опциональными X-ray/кросс-секционными вариациями. Базовая схема построена на Efficiency Nodes и глубинном ControlNet, а расширенные блоки используют Impact Pack для локального доработки проблемных зон и финального апскейла.

## Зависимости и кастомные узлы
Чтобы схема открылась без ошибок, установите следующие пакеты кастомных узлов:

- **Efficiency Nodes for ComfyUI** (`efficiency-nodes-comfyui`) — даёт узлы `Efficient Loader`, продвинутый `KSampler Adv.`, сценарий `HighRes-Fix Script` и `LoRA Stacker`.
- **ComfyUI-Impact-Pack** (в реестре значится как `comfyui-workflow-encrypt`) — содержит `FaceDetailer`, `Hand/Xray` детайлеры, масочные утилиты (`ImpactSimpleDetectorSEGS`, `ImpactDilateMask`, `ImpactGaussianBlurMask`, `ImpactConditionalBranch`, `ImpactIsNotEmptySEGS`) и финальный `easy hiresFix` апскейлер.
- **comfyui_controlnet_aux** — предобработчик `DepthAnythingV2Preprocessor` для глубинной карты.
- **ComfyUI_JPS-Nodes** — узел `SDXL Resolutions (JPS)` с пресетами разрешений.
- **rgthree-comfy** — узел качества жизни `Seed (rgthree)` для фиксации/случайного выбора зерна.

## Модели и ресурсы
Перед импортом убедитесь, что необходимые веса лежат в нужных каталогах ComfyUI:

- `ComfyUI/models/checkpoints/aozoraXLVpred_v01AlphaVpred.safetensors` — основной SDXL чекпоинт, загружается узлом `Efficient Loader`. При необходимости замените на свой чекпоинт и VAE.
- `ComfyUI/models/controlnet/noob_sdxl_controlnet_depth.safetensors` — Depth ControlNet, используется и в основном проходе, и в блоке HandDetailer с весом 0.5.
- `ComfyUI/custom_nodes/comfyui_controlnet_aux/weights/depth_anything_v2_vitl.pth` — вес модели DepthAnything v2 для генерации глубинной карты.
- `ComfyUI/models/sam/sam_vit_h_4b8939.pth` — SAM-модель, необходима детекторам Impact Pack для построения масок.
- `ComfyUI/custom_nodes/ComfyUI-Impact-Pack/models/ultralytics/bbox/hand_yolov8s.pt` — детектор рук.
- `ComfyUI/custom_nodes/ComfyUI-Impact-Pack/models/ultralytics/bbox/face_yolov8n_v2.pt` — детектор лиц.
- `ComfyUI/custom_nodes/ComfyUI-Impact-Pack/models/ultralytics/bbox/ntd11_anime_nsfw_segm_v4_x-ray.pt` и `..._cross-section.pt` — специализированные детекторы для X-ray и cross-section вариантов.
- `ComfyUI/models/upscale_models/4x-AnimeSharp.pth` — апскейлер, задействован и в `HighRes-Fix Script`, и в `easy hiresFix`.
- При желании добавить LoRA используйте узел `LoRA Stacker` (по умолчанию все слоты пусты).

## Импорт в ComfyUI
1. Убедитесь, что все кастомные узлы и модели установлены и ComfyUI перезапущен после копирования весов.
2. Сохраните файл `RindexWorflow.json` из репозитория в удобную папку.
3. В веб-интерфейсе ComfyUI нажмите `Load` → `Workflow` и выберите `RindexWorflow.json`.
4. Обновите позитивный промпт в узле `Ryuu_TokenCountTextBox` и при необходимости корректируйте негативный промпт/разрешение в `Efficient Loader` и `SDXL Resolutions (JPS)`.
5. Запустите выполнение (`Queue Prompt`). По умолчанию workflow генерирует портрет 832×1216 с последующим хайрез-проходом и финальным апскейлом.

## Опциональные блоки и рекомендации
### HandDetailer
Блок внутри зелёной группы использует `hand_yolov8s.pt` для поиска кистей, SAM для сегментации и `ImpactConditionalBranch`, чтобы запускать переработку только при наличии масок. Маски размягчаются (гаусс 10 px) и расширяются (`ImpactDilateMask` = 50), затем через `SetLatentNoiseMask` ограничивают пере-семплинг руками. Глубинный ControlNet подключён с весом 0.5 для сохранения формы. Рекомендации:
- Если руки не ловятся, понизьте порог детектора (стартовое значение 0.35); при ложных срабатываниях увеличьте его.
- Уменьшите/увеличьте дилатацию (50) и размытие, чтобы точнее ограничить область правки.
- При необходимости отключения можно обнулить вес ControlNet или выключить ветку, выставив нулевой выход у `ImpactConditionalBranch`.

### FaceDetailer
Основной FaceDetailer (фиолетовая группа) использует `face_yolov8n_v2.pt` и работает в тайлах 768 px с 35 шагами на DPM++ 3M SDE. Маска расширена на 16 пикселей с перышком 1.5, денойз установлен на 0.5 для мягкого уточнения кожи. Порог детекции выставлен на ~0.93 (IoU ~0.70). Рекомендации:
- Уменьшайте порог/IoU, если мелкие лица не обнаруживаются; повышайте при ложных срабатываниях.
- Регулируйте денойз (0.5) в диапазоне 0.35–0.6 в зависимости от желаемой агрессивности переработки.
- Маску можно сделать уже/шире, меняя параметры дилатации (16) и перышка (1.5), а также при необходимости подбирая CFG-скейл под свой стиль.

### Xray и Cross-Section Detailer
Голубая группа содержит два независимых детайлера с детекторами `ntd11_anime_nsfw_segm_v4_x-ray.pt` и `..._cross-section.pt`. Оба используют FaceDetailer с тайлом 512 px, 35 шагами, денойзом 0.5. Каждый детализатор выводит предпросмотр, поэтому их легко отключить, разорвав соответствующую связь или сменив вес детектора.
- Меняйте промпт внутри узлов FaceDetailer для желаемого эффекта X-ray/разреза.
- Настраивайте пороги детекторов (0.93/0.70 по умолчанию) под вашу выборку.
- Если эффекты не нужны, удалите/отключите конкретный FaceDetailer, не затрагивая остальную схему.

### Финальный апскейл
`easy hiresFix` выполняет финальный апскейл с помощью `4x-AnimeSharp`. По умолчанию увеличивает размер на 50 %, затем фиксирует выход в 1024×1024. При необходимости замените модель апскейла или измените режим масштабирования.

## Быстрый чек-лист
- ✅ Компоненты кастомных узлов установлены и обновлены.
- ✅ Все веса моделей лежат в нужных каталогах.
- ✅ Промпты и LoRA настроены под задачу.
- ✅ Опциональные детайлеры включены только там, где они действительно нужны.
