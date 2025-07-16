# Инструкция по использованию библиотеки SatelliteProcessor

## Содержание
1. [Установка и настройка](#установка-и-настройка)
2. [Основные концепции](#основные-концепции)
3. [Быстрый старт](#быстрый-старт)
4. [Детальное описание функций](#детальное-описание-функций)
5. [Параметры обработки](#параметры-обработки)
6. [Работа с индексами](#работа-с-индексами)
7. [Создание композитов](#создание-композитов)
8. [Типичные сценарии использования](#типичные-сценарии-использования)
9. [Советы по оптимизации](#советы-по-оптимизации)

## Установка и настройка

### Шаг 1: Сохранение библиотеки в GEE
1. Создайте новый репозиторий в Code Editor GEE
2. Назовите его `satellite-processing`
3. Скопируйте код библиотеки и сохраните как `satellite-processing.js`
4. Сделайте репозиторий публичным или приватным по необходимости

### Шаг 2: Импорт в проект
```javascript
var satLib = require('users/YOUR_USERNAME/satellite-processing:satellite-processing');
var SatelliteProcessor = satLib.SatelliteProcessor;
```

## Основные концепции

### Поддерживаемые спутники
- **Sentinel-2** (`SatelliteProcessor.SENTINEL2`)
- **Landsat 4/5** (`SatelliteProcessor.LANDSAT45`)
- **Landsat 8/9** (`SatelliteProcessor.LANDSAT89`)

### Ключевые функции
1. **getCleanCollection()** - получение очищенной коллекции снимков
2. **addIndices** - набор функций для добавления различных индексов
3. **composites** - функции для создания композитов

## Быстрый старт

### Минимальный пример
```javascript
// Определяем параметры
var roi = ee.Geometry.Rectangle([47.2, 53.9, 54.4, 56.7]);
var collection = SatelliteProcessor.getCleanCollection(
  SatelliteProcessor.SENTINEL2,
  roi,
  '2023-01-01',
  '2023-12-31'
);

// Создаем медианный композит
var median = collection.median();
Map.addLayer(median, {bands: ['B4', 'B3', 'B2'], min: 0, max: 0.3}, 'RGB');
```

### Пример с индексами
```javascript
// Получаем коллекцию с автоматическим расчетом индексов
var collectionWithIndices = SatelliteProcessor.getCleanCollection(
  SatelliteProcessor.SENTINEL2,
  roi,
  '2023-01-01',
  '2023-12-31',
  {
    indices: ['NDVI', 'NDRE', 'EVI2', 'NDWI']
  }
);

// Создаем композит и визуализируем
var composite = collectionWithIndices.median();
Map.addLayer(composite.select('NDVI'), {min: 0, max: 1, palette: ['red', 'yellow', 'green']}, 'NDVI');
Map.addLayer(composite.select('NDWI'), {min: -0.5, max: 0.5, palette: ['brown', 'white', 'blue']}, 'NDWI');
```

## Детальное описание функций

### getCleanCollection(satelliteType, roi, startDate, endDate, options)

Основная функция для получения предобработанной коллекции снимков.

**Параметры:**
- `satelliteType` (string) - тип спутника
- `roi` (ee.Geometry) - область интереса
- `startDate` (string) - начальная дата в формате 'YYYY-MM-DD'
- `endDate` (string) - конечная дата в формате 'YYYY-MM-DD'
- `options` (object) - дополнительные параметры (необязательно)

**Что делает функция:**
1. Фильтрует коллекцию по области и датам
2. Применяет масштабные коэффициенты (scale factors)
3. Маскирует облака и тени
4. Опционально: фильтрует по месяцам
5. Опционально: маскирует по NDVI
6. Опционально: маскирует по классификации

### Функции добавления индексов

#### sentinel2Vegetation(image)
Добавляет вегетационные индексы для Sentinel-2:
- NDVI, EVI2, TSAVI, NDRE

#### landsat45Soil(image)
Добавляет почвенные индексы для Landsat 4/5:
- EMBI, BITM, BIXS, BaI, DBSI, NDBaI, NDSoI, NSDS, RI4XS

#### landsat89Vegetation(image)
Добавляет вегетационные индексы для Landsat 8/9:
- AFRI2100, ARVI, EVI, MTVI2, NDMI, NDVI, NIRv, NMDI, OSAVI, sNIRvSWIR, WDRVI

### Функции создания композитов

#### median(collection, bands)
Создает медианный композит из коллекции.

#### statistics(collection, band)
Создает композит со статистиками (median, mean, max, stdDev) для указанного канала.

## Параметры обработки

### Объект options
```javascript
var options = {
  cloudThreshold: 0.85,        // Порог для Cloud Score (только S2)
  months: [5, 9],              // Фильтр по месяцам (null = все месяцы)
  ndviThreshold: 0.2,          // Маскировать пиксели с NDVI > значения
  classificationMask: image,   // ee.Image с классификацией
  classesToExclude: [1, 2],    // Классы для исключения
  indices: ['NDVI', 'NDRE']    // Список индексов для расчета (null = не считать)
};
```

## Работа с индексами

### Способы добавления индексов

#### 1. Автоматический расчет индексов при получении коллекции
```javascript
// Рассчитать конкретные индексы
var collection = SatelliteProcessor.getCleanCollection(
  SatelliteProcessor.SENTINEL2,
  roi, startDate, endDate,
  {
    indices: ['NDVI', 'NDRE', 'EVI2', 'NDMI']
  }
);

// Рассчитать все поддерживаемые индексы
var collectionAll = SatelliteProcessor.getCleanCollection(
  SatelliteProcessor.SENTINEL2,
  roi, startDate, endDate,
  {
    indices: SatelliteProcessor.supportedIndices.sentinel2.all
  }
);

// Рассчитать только вегетационные индексы
var collectionVeg = SatelliteProcessor.getCleanCollection(
  SatelliteProcessor.SENTINEL2,
  roi, startDate, endDate,
  {
    indices: SatelliteProcessor.supportedIndices.sentinel2.vegetation
  }
);
```

#### 2. Добавление индексов после получения коллекции
```javascript
// Использование предопределенных наборов
var s2WithVegIndices = collection.map(
  SatelliteProcessor.addIndices.sentinel2Vegetation
);

// Указание конкретных индексов
var s2WithCustomIndices = collection.map(function(img) {
  return SatelliteProcessor.addIndices.sentinel2Vegetation(
    img, 
    ['NDVI', 'NDRE', 'SAVI', 'MSAVI']
  );
});

// Добавление всех индексов
var s2WithAllIndices = collection.map(
  SatelliteProcessor.addIndices.sentinel2All
);
```

### Работа с категориями индексов

```javascript
// Получить список индексов по категории
var vegetationIndices = SatelliteProcessor.indices.getSupported(
  SatelliteProcessor.SENTINEL2, 
  'vegetation'
);

var waterIndices = SatelliteProcessor.indices.getSupported(
  SatelliteProcessor.LANDSAT89,
  'water'
);

var soilIndices = SatelliteProcessor.indices.getSupported(
  SatelliteProcessor.LANDSAT45,
  'soil'
);

var burnIndices = SatelliteProcessor.indices.getSupported(
  SatelliteProcessor.SENTINEL2,
  'burn'
);

// Получить все индексы для спутника
var allIndices = SatelliteProcessor.indices.getSupported(
  SatelliteProcessor.SENTINEL2
);
```

### Проверка поддержки индексов

```javascript
// Проверить поддержку конкретного индекса
var isSupported = SatelliteProcessor.indices.isSupported(
  SatelliteProcessor.SENTINEL2,
  'NDRE'
);

// Получить категорию индекса
var category = SatelliteProcessor.indices.getCategory(
  SatelliteProcessor.SENTINEL2,
  'NDVI'
); // вернет 'vegetation'

// Безопасное добавление индексов
var requestedIndices = ['NDVI', 'NDRE', 'CustomIndex', 'NDMI'];
var validIndices = requestedIndices.filter(function(idx) {
  return SatelliteProcessor.indices.isSupported(
    SatelliteProcessor.SENTINEL2, 
    idx
  );
});
```

### Пример добавления индексов к изображению
```javascript
// Для Sentinel-2
var s2WithIndices = collection.map(SatelliteProcessor.addIndices.sentinel2Vegetation);

// Для Landsat 8/9
var l89WithIndices = collection.map(SatelliteProcessor.addIndices.landsat89Vegetation);

// Для Landsat 4/5 (почвенные индексы)
var l45WithIndices = collection.map(SatelliteProcessor.addIndices.landsat45Soil);
```

### Создание собственных индексов
```javascript
var customIndex = collection.map(function(img) {
  // Для Sentinel-2
  var customNDVI = img.normalizedDifference(['B8', 'B4']).rename('customNDVI');
  return img.addBands(customNDVI);
});
```

## Создание композитов

### Простой медианный композит
```javascript
var median = SatelliteProcessor.composites.median(collection);
```

### Композит только для выбранных каналов
```javascript
var medianRGB = SatelliteProcessor.composites.median(
  collection, 
  ['B4', 'B3', 'B2']
);
```

### Статистические композиты
```javascript
// Получаем статистики для NDVI
var ndviStats = SatelliteProcessor.composites.statistics(
  collectionWithIndices, 
  'NDVI'
);
// Результат содержит: NDVI_median, NDVI_mean, NDVI_max, NDVI_stdDev
```

## Типичные сценарии использования

### Сценарий 1: Анализ изменений растительности
```javascript
// Определяем нужные индексы
var vegIndices = ['NDVI', 'NDRE', 'EVI2', 'SAVI', 'MSAVI'];

// Весна
var spring = SatelliteProcessor.getCleanCollection(
  SatelliteProcessor.SENTINEL2,
  roi, '2023-04-01', '2023-05-31',
  {
    months: [4, 5],
    indices: vegIndices
  }
);

// Лето
var summer = SatelliteProcessor.getCleanCollection(
  SatelliteProcessor.SENTINEL2,
  roi, '2023-07-01', '2023-08-31',
  {
    months: [7, 8],
    indices: vegIndices
  }
);

// Создаем медианы
var springMedian = SatelliteProcessor.composites.median(spring);
var summerMedian = SatelliteProcessor.composites.median(summer);

// Разница для всех индексов
vegIndices.forEach(function(idx) {
  var change = summerMedian.select(idx)
    .subtract(springMedian.select(idx))
    .rename(idx + '_change');
  
  Map.addLayer(change, {
    min: -0.3, max: 0.3, 
    palette: ['red', 'white', 'green']
  }, idx + ' Change');
});
```

### Сценарий 2: Анализ голых почв
```javascript
// Получаем все почвенные индексы
var soilIndices = SatelliteProcessor.indices.getSupported(
  SatelliteProcessor.LANDSAT45,
  'soil'
);

// Получаем снимки с низким NDVI и почвенными индексами
var baresoil = SatelliteProcessor.getCleanCollection(
  SatelliteProcessor.LANDSAT45,
  roi, '1985-01-01', '1995-12-31',
  {
    months: [4, 10],
    ndviThreshold: 0.2,
    indices: soilIndices
  }
);

// Создаем композит
var soilComposite = SatelliteProcessor.composites.median(baresoil);

// Визуализация разных почвенных индексов
['BITM', 'BSI', 'NDCSI', 'RI4XS'].forEach(function(idx) {
  Map.addLayer(
    soilComposite.select(idx),
    {min: -1, max: 1, palette: ['blue', 'white', 'red']},
    idx + ' Index'
  );
});
```

### Сценарий 3: Мониторинг леса с маскированием
```javascript
// Загружаем классификацию
var classification = ee.Image('projects/your-project/assets/classification');

// Получаем только лесные пиксели (класс 2)
var forest = SatelliteProcessor.getCleanCollection(
  SatelliteProcessor.LANDSAT89,
  roi, '2020-01-01', '2024-12-31',
  {
    classificationMask: classification,
    classesToExclude: [1, 3, 4, 5, 6, 7], // все кроме леса (2)
    indices: ['NDVI', 'EVI', 'LAI', 'SAVI', 'GNDVI']
  }
);

// Создаем композит
var forestComposite = SatelliteProcessor.composites.median(forest);
```

### Сценарий 4: Анализ водных объектов
```javascript
// Получаем все водные индексы для Sentinel-2
var waterIndices = SatelliteProcessor.indices.getSupported(
  SatelliteProcessor.SENTINEL2,
  'water'
);

print('Используемые водные индексы:', waterIndices);

// Получаем коллекцию с водными индексами
var waterCollection = SatelliteProcessor.getCleanCollection(
  SatelliteProcessor.SENTINEL2,
  roi, '2023-01-01', '2023-12-31',
  {
    months: [6, 7, 8], // летний период
    indices: waterIndices
  }
);

// Создаем композит
var waterComposite = SatelliteProcessor.composites.median(waterCollection);

// Анализируем основные водные индексы
['NDWI', 'MNDWI', 'AWEI_sh', 'SWI'].forEach(function(idx) {
  if (waterComposite.bandNames().contains(idx).getInfo()) {
    Map.addLayer(
      waterComposite.select(idx),
      {min: -0.5, max: 0.5, palette: ['brown', 'white', 'blue']},
      idx + ' Water Index'
    );
  }
});

// Создаем маску воды на основе MNDWI
var waterMask = waterComposite.select('MNDWI').gt(0);
Map.addLayer(waterMask.selfMask(), {palette: 'blue'}, 'Water Mask');
```

### Сценарий 5: Комплексный анализ с разными категориями индексов
```javascript
// Определяем индексы разных категорий
var analysisIndices = {
  vegetation: ['NDVI', 'EVI2', 'SAVI', 'LAI'],
  water: ['NDWI', 'MNDWI'],
  soil: ['BSI', 'NDCSI'],
  burn: ['NBR', 'NBR2']
};

// Объединяем все индексы
var allIndices = [];
Object.keys(analysisIndices).forEach(function(category) {
  allIndices = allIndices.concat(analysisIndices[category]);
});

// Проверяем поддержку индексов
var supportedIndices = allIndices.filter(function(idx) {
  return SatelliteProcessor.indices.isSupported(
    SatelliteProcessor.SENTINEL2, 
    idx
  );
});

print('Запрошено индексов:', allIndices.length);
print('Поддерживается:', supportedIndices.length);
print('Поддерживаемые индексы:', supportedIndices);

// Получаем коллекцию
var multiIndexCollection = SatelliteProcessor.getCleanCollection(
  SatelliteProcessor.SENTINEL2,
  roi, '2023-05-01', '2023-09-30',
  {
    indices: supportedIndices
  }
);

// Создаем композит
var multiIndexComposite = SatelliteProcessor.composites.median(multiIndexCollection);

// Экспортируем для дальнейшего анализа
Export.image.toAsset({
  image: multiIndexComposite,
  description: 'Multi_Index_Composite_2023',
  assetId: 'projects/your-project/assets/multi_index_2023',
  scale: 10,
  region: roi,
  maxPixels: 1e13
});
```

### Сценарий 6: Создание временных композитов
```javascript
var years = [2020, 2021, 2022, 2023, 2024];

years.forEach(function(year) {
  var yearCollection = SatelliteProcessor.getCleanCollection(
    SatelliteProcessor.SENTINEL2,
    roi,
    year + '-05-01',
    year + '-09-30',
    {months: [5, 6, 7, 8, 9]}
  );
  
  var composite = SatelliteProcessor.composites.median(yearCollection);
  
  Export.image.toAsset({
    image: composite,
    description: 'S2_Composite_' + year,
    assetId: 'projects/your-project/assets/S2_' + year,
    scale: 10,
    region: roi
  });
});
```

## Советы по оптимизации

### 1. Ограничивайте временной диапазон
Вместо обработки 5 лет сразу, разбейте на годовые интервалы:
```javascript
// Плохо
var allYears = SatelliteProcessor.getCleanCollection(
  SatelliteProcessor.SENTINEL2,
  roi, '2020-01-01', '2024-12-31'
);

// Хорошо
var year2023 = SatelliteProcessor.getCleanCollection(
  SatelliteProcessor.SENTINEL2,
  roi, '2023-01-01', '2023-12-31'
);
```

### 2. Используйте фильтр по месяцам
```javascript
// Только вегетационный период
var vegetation = SatelliteProcessor.getCleanCollection(
  SatelliteProcessor.SENTINEL2,
  roi, '2023-01-01', '2023-12-31',
  {months: [5, 9]} // май-сентябрь
);
```

### 3. Экспортируйте промежуточные результаты
```javascript
// Сохраняем очищенную коллекцию как ассет
var cleaned = SatelliteProcessor.getCleanCollection(...);
var composite = cleaned.median();

Export.image.toAsset({
  image: composite,
  description: 'cleaned_composite',
  assetId: 'projects/your-project/assets/cleaned_composite',
  scale: 10,
  region: roi
});
```

### 4. Оптимизация для больших территорий
```javascript
// Разбейте большую область на тайлы
var grid = roi.coveringGrid('EPSG:3857', 100000); // 100км тайлы

grid.aggregate_array('.geo').evaluate(function(tiles) {
  tiles.forEach(function(tile, i) {
    var tileGeom = ee.Geometry(tile);
    var collection = SatelliteProcessor.getCleanCollection(
      SatelliteProcessor.SENTINEL2,
      tileGeom,
      '2023-01-01', '2023-12-31'
    );
    // Обработка каждого тайла отдельно
  });
});
```

### 5. Мониторинг прогресса
```javascript
// Проверка количества снимков
var collection = SatelliteProcessor.getCleanCollection(...);
print('Количество снимков:', collection.size());
print('Первый снимок:', collection.first());
```

## Возможные проблемы и решения

### Проблема: "Too many pixels" при экспорте
**Решение:** Увеличьте масштаб или уменьшите область
```javascript
Export.image.toDrive({
  image: composite,
  scale: 30,  // увеличьте до 30, 100 или больше
  maxPixels: 1e13
});
```

### Проблема: Пустая коллекция
**Решение:** Проверьте параметры фильтрации
```javascript
var size = collection.size();
print('Размер коллекции:', size);

// Если 0, проверьте:
// 1. Правильность дат
// 2. Доступность снимков в регионе
// 3. Слишком строгие фильтры (cloudThreshold, ndviThreshold)
```

### Проблема: Недостаточно памяти
**Решение:** Обрабатывайте меньшими частями
```javascript
// Вместо всего года - по месяцам
var months = ee.List.sequence(1, 12);
var monthlyComposites = months.map(function(month) {
  var m = ee.Number(month);
  var collection = SatelliteProcessor.getCleanCollection(
    SatelliteProcessor.SENTINEL2,
    roi,
    '2023-01-01', '2023-12-31',
    {months: [m, m]}
  );
  return collection.median().set('month', m);
});
```

## Заключение

Библиотека SatelliteProcessor предоставляет унифицированный интерфейс для работы с различными спутниковыми системами в Google Earth Engine. Она автоматизирует рутинные операции предобработки и позволяет сосредоточиться на анализе данных.

### Ключевые возможности:
- Единый интерфейс для Sentinel-2, Landsat 4/5 и Landsat 8/9
- Автоматическая предобработка (облака, тени, масштабирование)
- Гибкая система расчета спектральных индексов
- Поддержка более 100 различных индексов
- Категоризация индексов (вегетация, вода, почва, гари)
- Встроенные функции создания композитов и статистик

### Преимущества новой версии:
- **Гибкость**: выбор конкретных индексов для расчета
- **Производительность**: расчет только необходимых индексов
- **Удобство**: автоматический расчет индексов при получении коллекции
- **Безопасность**: проверка поддержки индексов перед расчетом
- **Расширяемость**: легко добавлять новые индексы и функции

При возникновении вопросов обращайтесь к примерам использования или создавайте issue в репозитории проекта.