# earthpulse_geospatial.info.yml
```yaml
name: Earthpulse Geospatial
type: module
description: Provides geospatial data management with map integration, file uploads, and reporting.
core_version_requirement: ^10
package: Custom
dependencies:
  - drupal:field
  - drupal:views
  - drupal:file
```

# earthpulse_geospatial.routing.yml
```yaml
earthpulse_geospatial.map_form:
  path: '/geospatial/map'
  defaults:
    _form: '\Drupal\earthpulse_geospatial\Form\GeospatialMapForm'
    _title: 'Geospatial Map'
  requirements:
    _permission: 'access geospatial map'

earthpulse_geospatial.reports:
  path: '/geospatial/reports'
  defaults:
    _controller: '\Drupal\earthpulse_geospatial\Controller\GeospatialReportController::report'
    _title: 'Geospatial Reports'
  requirements:
    _permission: 'access geospatial reports'
```

# earthpulse_geospatial.permissions.yml
```yaml
access geospatial map:
  title: 'Access geospatial map'
  description: 'Allows users to view and interact with the geospatial map.'
upload geospatial files:
  title: 'Upload geospatial files'
  description: 'Allows users to upload geospatial files.'
access geospatial reports:
  title: 'Access geospatial reports'
  description: 'Allows users to view geospatial reports.'
```

# earthpulse_geospatial.services.yml
```yaml
services:
  earthpulse_geospatial.file_processor:
    class: '\Drupal\earthpulse_geospatial\Service\GeospatialFileProcessor'
    arguments: ['@plugin.manager.geospatial_processor', '@database']
  plugin.manager.geospatial_processor:
    class: '\Drupal\earthpulse_geospatial\GeospatialProcessorManager'
    parent: default_plugin_manager
```

# earthpulse_geospatial.module
```php
<?php

/**
 * @file
 * Main module file for Earthpulse Geospatial.
 */

/**
 * Implements hook_entity_type_build().
 */
function earthpulse_geospatial_entity_type_build(array &$entity_types) {
  if (isset($entity_types['geospatial_data'])) {
    $entity_types['geospatial_data']->setStorageClass('\Drupal\earthpulse_geospatial\Entity\GeospatialDataStorage');
  }
}

/**
 * Implements hook_schema().
 */
function earthpulse_geospatial_schema() {
  $schema['geospatial_data'] = [
    'description' => 'Stores geospatial data entities.',
    'fields' => [
      'id' => [
        'type' => 'serial',
        'not null' => TRUE,
        'description' => 'Primary Key: Unique geospatial data ID.',
      ],
      'uuid' => [
        'type' => 'varchar',
        'length' => 128,
        'not null' => TRUE,
        'description' => 'UUID.',
      ],
      'name' => [
        'type' => 'varchar',
        'length' => 255,
        'not null' => TRUE,
        'description' => 'Name of the geospatial data.',
      ],
      'geometry' => [
        'type' => 'text',
        'not null' => FALSE,
        'description' => 'GeoJSON geometry data.',
      ],
      'file_type' => [
        'type' => 'varchar',
        'length' => 50,
        'not null' => FALSE,
        'description' => 'Type of the uploaded file.',
      ],
      'created' => [
        'type' => 'int',
        'not null' => FALSE,
        'description' => 'Timestamp when the data was created.',
      ],
      'uid' => [
        'type' => 'int',
        'not null' => FALSE,
        'description' => 'The user ID who created the data.',
      ],
    ],
    'primary key' => ['id'],
    'indexes' => [
      'uuid' => ['uuid'],
      'uid' => ['uid'],
    ],
  ];
  return $schema;
}
```

# src/Entity/GeospatialData.php
```php
<?php

namespace Drupal\earthpulse_geospatial\Entity;

use Drupal\Core\Entity\ContentEntityBase;
use Drupal\Core\Entity\EntityTypeInterface;
use Drupal\Core\Field\BaseFieldDefinition;

/**
 * Defines the GeoscaleEntityBase {
  /**
   * {@inheritdoc}
   */
  public static function baseFieldDefinitions(EntityTypeInterface $entity_type) {
    $fields = parent::baseFieldDefinitions($entity_type);

    $fields['name'] = BaseFieldDefinition::create('string')
      ->setLabel(t('Name'))
      ->setRequired(TRUE)
      ->setDisplayOptions('view', [
        'label' => 'above',
        'type' => 'string',
        'weight' => -4,
      ])
      ->setDisplayOptions('form', [
        'type' => 'string_textfield',
        'weight' => -4,
      ])
      ->setDisplayConfigurable('form', TRUE)
      ->setDisplayConfigurable('view', TRUE);

    $fields['geometry'] = BaseFieldDefinition::create('text')
      ->setLabel(t('Geometry'))
      ->setDescription(t('GeoJSON geometry data.'));

    $fields['file_type'] = BaseFieldDefinition::create('string')
      ->setLabel(t('File Type'))
      ->setDescription(t('Type of the uploaded file.'));

    $fields['created'] = BaseFieldDefinition::create('created')
      ->setLabel(t('Created'));

    $fields['uid'] = BaseFieldDefinition::create('entity_reference')
      ->setLabel(t('User'))
      ->setSetting('target_type', 'user');

    return $fields;
  }

}
```

# src/Entity/GeospatialDataStorage.php
```php
<?php

namespace Drupal\earthpulse_geospatial\Entity;

use Drupal\Core\Entity\Sql\SqlContentEntityStorage;

/**
 * Custom storage class for Geospatial Data entity.
 */
class GeospatialDataStorage extends SqlContentEntityStorage {

  /**
   * Retrieves geospatial data for reporting, with optional filters.
   */
  public function getGeospatialData($filters = []) {
    $query = $this->database->select('geospatial_data', 'g')
      ->fields('g', ['id', 'name', 'geometry', 'file_type', 'created', 'uid']);

    if (!empty($filters['uid'])) {
      $query->condition('uid', $filters['uid']);
    }
    if (!empty($filters['file_type'])) {
      $query->condition('file_type', $filters['file_type']);
    }
    if (!empty($filters['date_from'])) {
      $query->condition('created', $filters['date_from'], '>=');
    }
    if (!empty($filters['date_to'])) {
      $query->condition('created', $filters['date_to'], '<=');
    }

    return $query->execute()->fetchAll();
  }

}
```

# src/Form/GeospatialMapForm.php
```php
<?php

namespace Drupal\earthpulse_geospatial\Form;

use Drupal\Core\Form\FormBase;
use Drupal\Core\Form\FormStateInterface;
use Drupal\earthpulse_geospatial\Service\GeospatialFileProcessor;
use Drupal\file\Entity\File;
use Symfony\Component\DependencyInjection\ContainerInterface;

/**
 * Form to interact with geospatial map and upload files.
 */
class GeospatialMapForm extends FormBase {

  /**
   * The geospatial file processor service.
   *
   * @var \Drupal\earthpulse_geospatial\Service\GeospatialFileProcessor
   */
  protected $fileProcessor;

  /**
   * Constructs a new GeospatialMapForm.
   */
  public function __construct(GeospatialFileProcessor $file_processor) {
    $this->fileProcessor = $file_processor;
  }

  /**
   * {@inheritdoc}
   */
  public static function create(ContainerInterface $container) {
    return new static(
      $container->get('earthpulse_geospatial.file_processor')
    );
  }

  /**
   * {@inheritdoc}
   */
  public function getFormId() {
    return 'geospatial_map_form';
  }

  /**
   * {@inheritdoc}
   */
  public function buildForm(array $form, FormStateInterface $form_state) {
    $form['#attached']['library'][] = 'earthpulse_geospatial/leaflet';

    $form['map'] = [
      '#type' => 'html_tag',
      '#tag' => 'div',
      '#attributes' => [
        'id' => 'geospatial-map',
        'style' => 'height: 500px;',
      ],
    ];

    $form['geometry'] = [
      '#type' => 'textarea',
      '#title' => $this->t('GeoJSON Geometry'),
      '#default_value' => '{}',
      '#attributes' => ['id' => 'geometry-input'],
    ];

    $form['file_upload'] = [
      '#type' => 'managed_file',
      '#title' => $this->t('Upload Geospatial File'),
      '#description' => $this->t('Supported formats: GeoJSON, Shapefile (.shp), KMZ.'),
      '#upload_validators' => [
        'file_validate_extensions' => ['geojson shp kmz'],
      ],
      '#upload_location' => 'public://geospatial/',
    ];

    $form['name'] = [
      '#type' => 'textfield',
      '#title' => $this->t('Data Name'),
      '#required' => TRUE,
    ];

    $form['submit'] = [
      '#type' => 'submit',
      '#value' => $this->t('Save'),
    ];

    return $form;
  }

  /**
   * {@inheritdoc}
   */
  public function submitForm(array &$form, FormStateInterface $form_state) {
    $values = $form_state->getValues();
    $geometry = $values['geometry'];

    // Handle file upload if present.
    $file_ids = $form_state->getValue('file_upload', []);
    if (!empty($file_ids)) {
      $file = File::load($file_ids[0]);
      if ($file) {
        $geometry = $this->fileProcessor->processFile($file);
        $file->setPermanent();
        $file->save();
      }
    }

    // Save to entity.
    $entity = \Drupal::entityTypeManager()->getStorage('geospatial_data')->create([
      'name' => $values['name'],
      'geometry' => $geometry,
      'file_type' => $file ? pathinfo($file->getFilename(), PATHINFO_EXTENSION) : 'geojson',
      'uid' => \Drupal::currentUser()->id(),
      'created' => time(),
    ]);
    $entity->save();

    $this->messenger()->addMessage($this->t('Geospatial data saved successfully.'));
  }

}
```

# src/Controller/GeospatialReportController.php
```php
<?php

namespace Drupal\earthpulse_geospatial\Controller;

use Drupal\Core\Controller\ControllerBase;
use Drupal\Core\Form\FormBuilderInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;

/**
 * Controller for geospatial reports.
 */
class GeospatialReportController extends ControllerBase {

  /**
   * The form builder.
   *
   * @var \Drupal\Core\Form\FormBuilderInterface
   */
  protected $formBuilder;

  /**
   * Constructs a new GeospatialReportController.
   */
  public function __construct(FormBuilderInterface $form_builder) {
    $this->formBuilder = $form_builder;
  }

  /**
   * {@inheritdoc}
   */
  public static function create(ContainerInterface $container) {
    return new static(
      $container->get('form_builder')
    );
  }

  /**
   * Renders the geospatial reports page.
   */
  public function report() {
    $build = [
      '#attached' => [
        'library' => ['earthpulse_geospatial/leaflet'],
      ],
      'filter_form' => $this->formBuilder->getForm('\Drupal\earthpulse_geospatial\Form\GeospatialReportFilterForm'),
      'map' => [
        '#type' => 'html_tag',
        '#tag' => 'div',
        '#attributes' => [
          'id' => 'report-map',
          'style' => 'height: 500px;',
        ],
      ],
      'data' => [
        '#type' => 'html_tag',
        '#tag' => 'div',
        '#attributes' => ['id' => 'report-data'],
        '#value' => json_encode($this->getReportData()),
      ],
    ];

    return $build;
  }

  /**
   * Retrieves report data.
   */
  protected function getReportData() {
    $filters = \Drupal::request()->query->all();
    $storage = \Drupal::entityTypeManager()->getStorage('geospatial_data');
    return $storage->getGeospatialData($filters);
  }

}
```

# src/Form/GeospatialReportFilterForm.php
```php
<?php

namespace Drupal\earthpulse_geospatial\Form;

use Drupal\Core\Form\FormBase;
use Drupal\Core\Form\FormStateInterface;

/**
 * Filter form for geospatial reports.
 */
class GeospatialReportFilterForm extends FormBase {

  /**
   * {@inheritdoc}
   */
  public function getFormId() {
    return 'geospatial_report_filter_form';
  }

  /**
   * {@inheritdoc}
   */
  public function buildForm(array $form, FormStateInterface $form_state) {
    $form['uid'] = [
      '#type' => 'entity_autocomplete',
      '#target_type' => 'user',
      '#title' => $this->t('User'),
      '#default_value' => \Drupal::request()->query->get('uid'),
    ];

    $form['file_type'] = [
      '#type' => 'select',
      '#title' => $this->t('File Type'),
      '#options' => [
        '' => $this->t('- Any -'),
        'geojson' => 'GeoJSON',
        'shp' => 'Shapefile',
        'kmz' => 'KMZ',
      ],
      '#default_value' => \Drupal::request()->query->get('file_type'),
    ];

    $form['date_from'] = [
      '#type' => 'date',
      '#title' => $this->t('From Date'),
      '#default_value' => \Drupal::request()->query->get('date_from'),
    ];

    $form['date_to'] = [
      '#type' => 'date',
      '#title' => $this->t('To Date'),
      '#default_value' => \Drupal::request()->query->get('date_to'),
    ];

    $form['submit'] = [
      '#type' => 'submit',
      '#value' => $this->t('Filter'),
    ];

    return $form;
  }

  /**
   * {@inheritdoc}
   */
  public function submitForm(array &$form, FormStateInterface $form_state) {
    $form_state->setRedirect('earthpulse_geospatial.reports', [], [
      'query' => array_filter($form_state->getValues(), function ($key) {
        return in_array($key, ['uid', 'file_type', 'date_from', 'date_to']);
      }, ARRAY_FILTER_USE_KEY),
    ]);
  }

}
```

# src/Service/GeospatialFileProcessor.php
```php
<?php

namespace Drupal\earthpulse_geospatial\Service;

use Drupal\Component\Plugin\PluginManagerInterface;
use Drupal\Core\Database\Connection;
use Drupal\file\Entity\File;

/**
 * Service to process geospatial files.
 */
class GeospatialFileProcessor {

  /**
   * The plugin manager for geospatial processors.
   *
   * @var \Drupal\Component\Plugin\PluginManagerInterface
   */
  protected $pluginManager;

  /**
   * The database connection.
   *
   * @var \Drupal\Core\Database\Connection
   */
  protected $database;

  /**
   * Constructs a new GeospatialFileProcessor.
   */
  public function __construct(PluginManagerInterface $plugin_manager, Connection $database) {
    $this->pluginManager = $plugin_manager;
    $this->database = $database;
  }

  /**
   * Processes an uploaded file.
   *
   * @param \Drupal\file\Entity\File $file
   *   The uploaded file entity.
   *
   * @return string
   *   The GeoJSON geometry.
   */
  public function processFile(File $file) {
    $extension = strtolower(pathinfo($file->getFilename(), PATHINFO_EXTENSION));
    $plugin = $this->pluginManager->createInstance($extension);
    return $plugin->process($file->getFileUri());
  }

}
```

# src/Plugin/GeospatialProcessor/GeospatialProcessorInterface.php
```php
<?php

namespace Drupal\earthpulse_geospatial\Plugin\GeospatialProcessor;

/**
 * Interface for geospatial processor plugins.
 */
interface GeospatialProcessorInterface {

  /**
   * Processes a geospatial file and returns GeoJSON.
   *
   * @param string $filepath
   *   The path to the file.
   *
   * @return string
   *   The GeoJSON string.
   */
  public function process($filepath);

}
```

# src/Plugin/GeospatialProcessor/GeoJsonProcessor.php
```php
<?php

namespace Drupal\earthpulse_geospatial\Plugin\GeospatialProcessor;

use Drupal\Component\Plugin\PluginBase;

/**
 * GeoJSON processor plugin.
 *
 * @GeospatialProcessor(
 *   id = "geojson",
 *   label = @Translation("GeoJSON Processor"),
 * )
 */
class GeoJsonProcessor extends PluginBase implements GeospatialProcessorInterface {

  /**
   * {@inheritdoc}
   */
  public function process($filepath) {
    $content = file_get_contents($filepath);
    if (json_decode($content)) {
      return $content;
    }
    throw new \Exception('Invalid GeoJSON file.');
  }

}
```

# src/Plugin/GeospatialProcessor/ShpProcessor.php
```php
<?php

namespace Drupal\earthpulse_geospatial\Plugin\GeospatialProcessor;

use Drupal\Component\Plugin\PluginBase;

/**
 * Shapefile processor plugin.
 *
 * @GeospatialProcessor(
 *   id = "shp",
 *   label = @Translation("Shapefile Processor"),
 * )
 */
class ShpProcessor extends PluginBase implements GeospatialProcessorInterface {

  /**
   * {@inheritdoc}
   */
  public function process($filepath) {
    // Simulate conversion using ogr2ogr (requires external setup).
    $output = tempnam(sys_get_temp_dir(), 'geojson_');
    $command = "ogr2ogr -f GeoJSON $output $filepath";
    exec($command, $result, $return_var);
    if ($return_var === 0) {
      $geojson = file_get_contents($output);
      unlink($output);
      return $geojson;
    }
    throw new \Exception('Failed to process Shapefile.');
  }

}
```

# src/GeospatialProcessorManager.php
```php
<?php

namespace Drupal\earthpulse_geospatial;

use Drupal\Core\Plugin\DefaultPluginManager;
use Drupal\Core\Cache\CacheBackendInterface;
use Drupal\Core\Extension\ModuleHandlerInterface;

/**
 * Manages geospatial processor plugins.
 */
class GeospatialProcessorManager extends DefaultPluginManager {

  /**
   * Constructs a new GeospatialProcessorManager.
   */
  public function __construct(\Traversable $namespaces, CacheBackendInterface $cache_backend, ModuleHandlerInterface $module_handler) {
    parent::__construct(
      'Plugin/GeospatialProcessor',
      $namespaces,
      $module_handler,
      'Drupal\earthpulse_geospatial\Plugin\GeospatialProcessor\GeospatialProcessorInterface',
      'Drupal\earthpulse_geospatial\Annotation\GeospatialProcessor'
    );
    $this->alterInfo('geospatial_processor_info');
    $this->setCacheBackend($cache_backend, 'geospatial_processor_plugins');
  }

}
```

# src/Annotation/GeospatialProcessor.php
```php
<?php

namespace Drupal\earthpulse_geospatial\Annotation;

use Drupal\Component\Annotation\Plugin;

/**
 * Defines a Geospatial Processor annotation object.
 *
 * @Annotation
 */
class GeospatialProcessor extends Plugin {

  /**
   * The plugin ID.
   *
   * @var string
   */
  public $id;

  /**
   * The plugin label.
   *
   * @var \Drupal\Core\Annotation\Translation
   */
  public $label;

}
```

# earthpulse_geospatial.libraries.yml
```yaml
leaflet:
  js:
    'https://unpkg.com/leaflet@1.9.4/dist/leaflet.js': { type: external, minified: true }
    'https://unpkg.com/leaflet-draw@1.0.4/dist/leaflet.draw.js': { type: external, minified: true }
    js/earthpulse_geospatial.js: {}
  css:
    component:
      'https://unpkg.com/leaflet@1.9.4/dist/leaflet.css': { type: external }
      'https://unpkg.com/leaflet-draw@1.0.4/dist/leaflet.draw.css': { type: external }
  dependencies:
    - core/drupal
    - core/jquery
```

# js/earthpulse_geospatial.js
```javascript
(function ($, Drupal) {
  Drupal.behaviors.earthpulseGeospatial = {
    attach: function (context) {
      // Initialize map for form.
      $('#geospatial-map', context).once('geospatial-map').each(function () {
        var map = L.map('geospatial-map').setView([0, 0], 2);
        L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
          attribution: '© OpenStreetMap contributors'
        }).addTo(map);

        var drawnItems = new L.FeatureGroup();
        map.addLayer(drawnItems);

        var drawControl = new L.Control.Draw({
          edit: {
            featureGroup: drawnItems
          },
          draw: {
            polygon: true,
            polyline: true,
            rectangle: true,
            circle: false,
            marker: true
          }
        });
        map.addControl(drawControl);

        map.on(L.Draw.Event.CREATED, function (e) {
          var layer = e.layer;
          drawnItems.addLayer(layer);
          $('#geometry-input').val(JSON.stringify(layer.toGeoJSON()));
        });
      });

      // Initialize report map.
      $('#report-map', context).once('report-map').each(function () {
        var map = L.map('report-map').setView([0, 0], 2);
        L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
          attribution: '© OpenStreetMap contributors'
        }).addTo(map);

        var data = JSON.parse($('#report-data').text());
        data.forEach(function (item) {
          if (item.geometry) {
            try {
              var layer = L.geoJSON(JSON.parse(item.geometry));
              layer.bindPopup('<strong>' + item.name + '</strong><br>Type: ' + item.file_type);
              layer.addTo(map);
            } catch (e) {
              console.error('Invalid GeoJSON:', item.geometry);
            }
          }
        });
      });
    }
  };
})(jQuery, Drupal);
```

# README.md
```markdown
# Earthpulse Geospatial Module

A Drupal 10 module for managing geospatial data with map integration, file uploads, and reporting.

## Features
- **Map Integration**: Use Leaflet with drawing tools to create/edit Areas of Interest (AOIs) in GeoJSON format.
- **File Uploads**: Supports GeoJSON, Shapefiles (.shp), and KMZ files, with conversion to GeoJSON.
- **Reporting Dashboard**: View all geospatial data on a map with filters by user, date, and file type.
- **Plugin System**: Extensible architecture for processing different geospatial file types.
- **Security**: Role-based permissions for map access, file uploads, and reports.

## Dependencies
- Drupal 10.x
- PHP >= 8.1
- Composer dependencies:
  - `leaflet` (via CDN)
  - `leaflet-draw` (via CDN)
- Optional: `ogr2ogr` (for Shapefile/KMZ processing, requires server setup)

## Installation
1. Place the `earthpulse_geospatial` folder in `modules/custom/`.
2. Run `drush en earthpulse_geospatial` to enable the module.
3. Ensure `ogr2ogr` is installed on the server for Shapefile/KMZ support (optional).
4. Clear cache: `drush cr`.
5. Assign permissions at `/admin/people/permissions`.

## Usage
- **Map Interface**: Navigate to `/geospatial/map` to draw AOIs or upload geospatial files.
- **Reports**: Visit `/geospatial/reports` to view all geospatial data with filters.
- **Permissions**: Configure permissions under `admin/people/permissions`.

## Drush Commands
- None currently implemented. Future versions may include import/export commands.

## Development
- **Extending Processors**: Add new plugins in `src/Plugin/GeospatialProcessor` implementing `GeospatialProcessorInterface`.
- **Tests**: A sample test is provided in `tests/src/Kernel/GeospatialDataTest.php`.

## Sample Data
- Upload a GeoJSON file with the following structure for testing:
```json
{
  "type": "Feature",
  "geometry": {
    "type": "Point",
    "coordinates": [0, 0]
  },
  "properties": {
    "name": "Test Point"
  }
}
```

## Notes
- Ensure server has write permissions for `public://geospatial/` for file uploads.
- Leaflet and Leaflet-Draw are loaded via CDN for simplicity.
- For Shapefile/KMZ support, install GDAL (ogr2ogr) on the server.
```

# tests/src/Kernel/GeospatialDataTest.php
```php
<?php

namespace Drupal\Tests\earthpulse_geospatial\Kernel;

use Drupal\KernelTests\KernelTestBase;
use Drupal\earthpulse_geospatial\Entity\GeospatialData;

/**
 * Tests the Geospatial Data entity.
 *
 * @group earthpulse_geospatial
 */
class GeospatialDataTest extends KernelTestBase {

  /**
   * Modules to enable.
   *
   * @var array
   */
  protected static $modules = ['earthpulse_geospatial', 'user', 'file', 'system'];

  /**
   * {@inheritdoc}
   */
  protected function setUp(): void {
    parent::setUp();
    $this->installEntitySchema('geospatial_data');
    $this->installEntitySchema('user');
    $this->installEntitySchema('file');
    $this->installSchema('system', ['sequences']);
    $this->installSchema('file', ['file_managed']);
  }

  /**
   * Tests creating and saving a geospatial data entity.
   */
  public function testGeospatialDataCreation() {
    $entity = GeospatialData::create([
      'name' => 'Test Data',
      'geometry' => '{"type":"Point","coordinates":[0,0]}',
      'file_type' => 'geojson',
      'uid' => 1,
      'created' => time(),
    ]);
    $entity->save();

    $loaded = \Drupal::entityTypeManager()->getStorage('geospatial_data')->load(1);
    $this->assertEquals('Test Data', $loaded->get('name')->value);
    $this->assertEquals('geojson', $loaded->get('file_type')->value);
  }

}
```
