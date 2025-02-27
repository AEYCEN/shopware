# 6.6.0.0
## Introduced in 6.5.8.0

## Storefront async JavaScript and all.js removal

With the upcoming major version v6.6.0 we want to get rid of the `all.js` in the Storefront and also allow async JavaScript with dynamic imports.
Our current webpack compiling for JavaScript alongside the `all.js` does not consider asynchronous imports.

### New distribution of App/Plugin "dist" JavaScript

The merging of your App/Plugin JavaScript into an `all.js` will no longer take place. Each App/Plugin will get its own JavaScript served by a separate `<script>` tag instead.
Essentially, all JavaScript inside your "dist" folder (`ExampleApp/src/Resources/app/storefront/dist/storefront/js`) will be distributed into the `public/theme` directory as it is.
Each App/Plugin will get a separate subdirectory which matches the App/Plugin technical name as snake-case, for example `public/theme/<theme-hash>/js/example-app/`.

This subdirectory will be added automatically during `composer build:js:storefront`. Please remove outdated generated JS files from the old location from your "dist" folder.
Please also include all additional JS files which might have been generated due to dynamic imports in your release:

Before:
```
└── custom/apps/
    └── ExampleApp/src/Resources/app/storefront/dist/storefront/js/
        └── example-app.js
```

After:
```
└── custom/apps/
    └── ExampleApp/src/Resources/app/storefront/dist/storefront/js/
        ├── example-app.js         <-- OLD: Will be ignored (but should be removed for theme:compile)
        └── example-app/           <-- NEW: Please include everything in this folder in the release
            ├── example-app.js     
            ├── async-example-1.js 
            └── async-example-2.js 
```

The distributed version in `/public/theme/<theme-hash>/js/` will look like below.

**Just to illustrate, you don't need to change anything manually here!**

Before:
```
└── public/theme/
    └── 6c7abe8363a0dfdd16929ca76c02aa35/
        ├── css/
        │   └── all.css
        └── js/
            └── all.js  
```

After:
```
└── public/theme/
    └── 6c7abe8363a0dfdd16929ca76c02aa35/
        ├── css/
        │   └── all.css
        └── js/
            ├── storefront/
            │   ├── storefront.js (main bundle of "storefront", generates <script>)
            │   ├── cross-selling_plugin.js
            │   └── listing_plugin.js
            └── example-app/
                ├── example-app (main bundle of "my-listing", generates <script>)
                ├── async-example-1.js
                └── async-example-2.js
```

### File path pattern for scripts in theme.json file
If the script file does not match the new file path pattern, it will be **ignored** (during getThemeScripts in Storefront, not during theme:compile).

Example for a Theme called MyOldTheme (theme.json)
```json
...
"script": [
  "@Storefront",
  "@AnotherTheme",
  "app/storefront/dist/storefront/js/my-old-theme.js", // This file will be ignored (structure before 6.6)
  "app/storefront/dist/storefront/js/my-old-theme/my-old-theme.js", // This file will be used (new structure)
],
...
```
We need to ignore the old files for multiple reasons. The main reason is that the old files are not compatible with the new async JavaScript and dynamic imports. Second it would throw an error for all themes that do not update their theme.json file.

### Re-compile your JavaScript

Because of the changes in the JavaScript compiling process and dynamic imports, it is not possible to have pre-compiled JavaScript (`ExampleApp/src/Resources/app/storefront/dist/storefront/js`)
to be cross-compatible with the current major lane v6.5.0 and v6.6.0 at the same time.

Therefore, we recommend to release a new App/Plugin version which is compatible with v6.6.0 onwards.
The JavaScript for the Storefront can be compiled as usual using the composer script `composer build:js:storefront`.

**The App/Plugin entry point for JS `main.js` and the general way to compile the JS remains the same!**

Re-compiling your App/Plugin is a good starting point to ensure compatibility.
If your App/Plugin mainly adds new JS-Plugins and does not override existing JS-Plugins, chances are that this is all you need to do in order to be compatible.

### JavaScript build separation of apps/plugin with webpack MultiCompiler

With 6.6 we use webpack [MultiCompiler](https://webpack.js.org/api/node/#multicompiler) to build the default storefront as well as apps and plugins.
Each app/plugin will generate its own webpack config in the background and will be built in a separate build process to enhance JS-bundle stability.

You can still extend the webpack config of the default storefront with your own config like in 6.5, for example to add a new alias.
Due to the build process separation, your modified webpack config will only take effect in your current app/plugin but will no longer effect other apps/plugins.

Let's imagine two apps "App1" and "App2". "App1" is now extending the webpack config with a custom alias. 
In the example below, your will have access to all "alias" from the default storefront, as well as the additional alias "ExampleAlias" in "App1":

```js
// App 1 webpack config
// custom/apps/App1/Resources/app/storefront/build/webpack.config.js
module.exports = function (params) {
    return {
        resolve: {
            alias: {
                // The alias "ExampleAlias" can only be used within App1
                ExampleAlias: `${params.basePath}/Resources/app/storefront/src/example-dir`,
            }
        }
    };
};
```

Now the alias can be used within "App1":
```js
// custom/apps/App1/Resources/app/storefront/src/main.js
import MyComponent from 'ExampleAlias/example-module'; // <-- ✅ Can be resolved
```

If the alias is used within "App2", you will get an error because the import cannot be resolved:
```js
// custom/apps/App2/Resources/app/storefront/src/main.js
import MyComponent from 'ExampleAlias/example-module'; // <-- ❌ Cannot be resolved
```

If you need the alias `ExampleAlias` or another config from "App1", you need to explicitly add the alias to "App2".
Apps/plugins should no longer be able to influence each other during the build process for stability reasons.
Your App/plugins webpack config only inherits the core webpack config but no other webpack configs.

### Registering async JS-plugins (optional)

To prevent all JS-plugins from being present on every page, we will offer the possibility to fetch the JS-plugins on-demand.
This is done by the `PluginManager` which determines if the selector from `register()` is present in the current document. Only if this is the case the JS-plugin will be fetched.

The majority of the platform Storefront JS-plugin will be changed to async.

**The general API to register JS-plugin remains the same!**

If you pass an arrow function with a dynamic import instead of a normal import,
your JS-plugin will be async and also generate an additional `.js` file in your `/dist` folder.

Before:
```js
import ExamplePlugin from './plugins/example.plugin';

window.PluginManager.register('Example', ExamplePlugin, '[data-example]');
```
After:
```js
window.PluginManager.register('Example', () => import('./plugins/example.plugin'), '[data-example]');
```

The "After" example above will generate:
```
└── custom/apps/
    └── ExampleApp/src/Resources/app/storefront/dist/storefront/js/
        └── example-app/           
            ├── example-app.js                 <-- The main app JS-bundle
            └── src_plugins_example_plugin.js  <-- Auto generated by the dynamic import
```

### Override async JS-plugins

If a platform Storefront plugin is async, the override class needs to be async as well.

Before:
```js
import MyListingExtensionPlugin from './plugin-extensions/listing/my-listing-extension.plugin';

window.PluginManager.override(
    'Listing', 
    MyListingExtensionPlugin, 
    '[data-listing]'
);
```
After:
```js
window.PluginManager.override(
    'Listing', 
    () => import('./plugin-extensions/listing/my-listing-extension.plugin'),
    '[data-listing]',
);
```

### Async plugin initialization with `PluginManager.initializePlugins()` and `PluginManager.initializePlugin()`

* The method `PluginManager.initializePlugins()` is now async and will return a Promise because it also downloads all async JS-plugins before their initialization.
* The method `PluginManager.initializePlugin()` to initialize a single JS-plugin is now async as well and will download the single plugin if was not downloaded beforehand.

If you need access to newly created JS-Plugin instances (for example after a dynamic DOM-update with new JS-Plugin selectors), you need to wait for the Promise to resolve.

Before:
```js
/**
 * Example scenario:
 * 1. A dynamic DOM update via JavaScript (e.g. Ajax) adds selector "[data-form-ajax-submit]"
 * 2. PluginManager.initializePlugins() intializes Plugin "FormAjaxSubmit" because a new selector is present.
 * 3. You need access to the Plugin instance of "FormAjaxSubmit" directly after PluginManager.initializePlugins().
 */
window.PluginManager.initializePlugins();

const FormAjaxSubmitInstance = window.PluginManager.getPluginInstanceFromElement(someElement, 'FormAjaxSubmit');
// ... does something with "FormAjaxSubmitInstance"
```

After:
```js
/**
 * Example scenario:
 * 1. A dynamic DOM update via JavaScript (e.g. Ajax) adds selector "[data-form-ajax-submit]"
 * 2. PluginManager.initializePlugins() intializes Plugin "FormAjaxSubmit" because a new selector is present.
 * 3. You need access to the Plugin instance of "FormAjaxSubmit" directly after PluginManager.initializePlugins().
 */
window.PluginManager.initializePlugins().then(() => {
    const FormAjaxSubmitInstance = window.PluginManager.getPluginInstanceFromElement(someElement, 'FormAjaxSubmit');
    // ... does something with "FormAjaxSubmitInstance"
});
```

If you don't need direct access to newly created JS-plugin instances via `getPluginInstanceFromElement()`, and you only want to "re-init" all JS-plugins,
you do not need to wait for the Promise of `initializePlugins()` or `initializePlugin()` because `initializePlugins()` and `initializePlugin()` already download and initialize the JS-plugins.

### Avoid import from PluginManager

Because the PluginManager is a singleton class which also assigns itself to the `window` object,
it should be avoided to import the PluginManager. It can lead to unintended side effects.

Use the existing `window.PluginManager` instead.

Before:
```js
import PluginManager from 'src/plugin-system/plugin.manager';

PluginManager.getPluginInstances('SomePluginName');
```
After:
```js
window.PluginManager.getPluginInstances('SomePluginName');
```

### Avoid import from Plugin base class

The import of the `Plugin` class can lead to code-duplication of the Plugin class in every App/Plugin.

Use `window.PluginBaseClass` instead.

Before:
```js
import Plugin from 'src/plugin-system/plugin.class';

export default class MyPlugin extends Plugin {
    // Plugin code...
};
```
After:
```js
export default class MyPlugin extends window.PluginBaseClass {
    // Plugin code...
};
```

## Removal of static product detail page templates

The deprecated template `src/Storefront/Resources/views/storefront/page/product-detail/index.html.twig` was removed and replaced by configurable product detail CMS pages.
Please use the template `src/Storefront/Resources/views/storefront/page/content/product-detail.html.twig` instead.

This also applies to the sub-templates of the product detail page. From now on, CMS components are used instead.
The old templates from `/page/product-detail` will no longer be used when a product detail page is rendered. 
The default product detail page CMS layout will be used, if no other layout is configured in the administration.

| Old                                                                           | New                                                                                  |
|-------------------------------------------------------------------------------|--------------------------------------------------------------------------------------|
| Resources/views/storefront/page/product-detail/tabs.html.twig                 | Resources/views/storefront/element/cms-element-product-description-reviews.html.twig |
| Resources/views/storefront/page/product-detail/description.html.twig          | Resources/views/storefront/component/product/description.html.twig                   |
| Resources/views/storefront/page/product-detail/properties.html.twig           | Resources/views/storefront/component/product/properties.html.twig                    |
| Resources/views/storefront/page/product-detail/headline.html.twig             | Resources/views/storefront/element/cms-element-product-name.html.twig                |
| Resources/views/storefront/page/product-detail/configurator.html.twig         | Resources/views/storefront/component/buy-widget/configurator.html.twig               |
| Resources/views/storefront/page/product-detail/buy-widget.html.twig           | Resources/views/storefront/component/buy-widget/buy-widget.html.twig                 |
| Resources/views/storefront/page/product-detail/buy-widget-price.html.twig     | Resources/views/storefront/component/buy-widget/buy-widget-price.html.twig           |
| Resources/views/storefront/page/product-detail/buy-widget-form.html.twig      | Resources/views/storefront/component/buy-widget/buy-widget-form.html.twig            |
| Resources/views/storefront/page/product-detail/review/review.html.twig        | Resources/views/storefront/component/review/review.html.twig                         |
| Resources/views/storefront/page/product-detail/review/review-form.html.twig   | Resources/views/storefront/component/review/review-form.html.twig                    |
| Resources/views/storefront/page/product-detail/review/review-item.html.twig   | Resources/views/storefront/component/review/review-item.html.twig                    |
| Resources/views/storefront/page/product-detail/review/review-login.html.twig  | Resources/views/storefront/component/review/review-login.html.twig                   |
| Resources/views/storefront/page/product-detail/review/review-widget.html.twig | Resources/views/storefront/component/review/review-widget.html.twig                  |
| Resources/views/storefront/page/product-detail/cross-selling/tabs.html.twig   | Resources/views/storefront/element/cms-element-cross-selling.html.twig               |

## Introduced in 6.5.7.0
## New media url generator and path strategy
* Removed deprecated `UrlGeneratorInterface` interface, use `AbstractMediaUrlGenerator` instead to generate the urls for media entities
* Removed deprecated `AbstractPathNameStrategy` abstract class, use `AbstractMediaPathStrategy` instead to implement own strategies

```php
<?php 

namespace Examples;

use Shopware\Core\Content\Media\Core\Application\AbstractMediaUrlGenerator;use Shopware\Core\Content\Media\Core\Params\UrlParams;use Shopware\Core\Content\Media\MediaCollection;use Shopware\Core\Content\Media\MediaEntity;use Shopware\Core\Content\Media\Pathname\UrlGeneratorInterface;

class BeforeChange
{
    private UrlGeneratorInterface $urlGenerator;
    
    public function foo(MediaEntity $media) 
    {
        $relative = $this->urlGenerator->getRelativeMediaUrl($media);
        
        $absolute = $this->urlGenerator->getAbsoluteMediaUrl($media);
    }
    
    public function bar(MediaThumbnailEntity $thumbnail) 
    {
        $relative = $this->urlGenerator->getRelativeThumbnailUrl($thumbnail);
        
        $absolute = $this->urlGenerator->getAbsoluteThumbnailUrl($thumbnail);
    }
}

class AfterChange
{
    private AbstractMediaUrlGenerator $generator;
    
    public function foo(MediaEntity $media) 
    {
        $relative = $media->getPath();

        $urls = $this->generator->generate([UrlParams::fromMedia($media)]);
        
        $absolute = $urls[0];
    }
    
    public function bar(MediaThumbnailEntity $thumbnail) 
    {
        // relative is directly stored at the entity
        $relative = $thumbnail->getPath();
        
        // path generation is no more entity related, you could also use partial entity loading and you can also call it in batch, see below
        $urls = $this->generator->generate([UrlParams::fromMedia($media)]);
        
        $absolute = $urls[0];
    }
    
    public function batch(MediaCollection $collection) 
    {
        $params = [];
        
        foreach ($collection as $media) {
            $params[$media->getId()] = UrlParams::fromMedia();
            
            foreach ($media->getThumbnails() as $thumbnail) {
                $params[$thumbnail->getId()] = UrlParams::fromThumbnail($thumbnail);
            }
        }
        
        $urls = $this->generator->generate($paths);

        // urls is a flat list with {id} => {url} for media and also for thumbnails        
    }
}
```
## New custom fields mapping event

* Previously the event `ElasticsearchProductCustomFieldsMappingEvent` is dispatched when create new ES index so you can add your own custom fields mapping.
* We replaced the event with a new event `Shopware\Elasticsearch\Event\ElasticsearchCustomFieldsMappingEvent`, this provides a better generic way to add custom fields mapping

```php
class ExampleCustomFieldsMappingEventSubscriber implements EventSubscriberInterface {

    public static function getSubscribedEvents(): array
    {
        return [
            ElasticsearchCustomFieldsMappingEvent::class => 'addCustomFieldsMapping',
        ];
    }

    public function addCustomFieldsMapping(ElasticsearchCustomFieldsMappingEvent $event): void 
    {
        if ($event->getEntity() === 'product') {
            $event->setMapping('productCfFoo', CustomFieldTypes::TEXT);
        }

        if ($event->getEntity() === 'category') {
            $event->setMapping('categoryCfFoo', CustomFieldTypes::TEXT);
        }
        // ...
    }
}
```

## Adding sugar syntax for ES Definition

We added new utility classes to make creating custom ES definition look simpler

In this example, assuming you have a custom ES definition with `name` & `description` fields are translatable fields:

```php
<?php declare(strict_types=1);

namespace Shopware\Commercial\AdvancedSearch\Domain\Indexing\ElasticsearchDefinition\Manufacturer;

use Shopware\Core\Framework\Context;
use Shopware\Core\Framework\DataAbstractionLayer\Dbal\SqlHelper;
use Shopware\Core\Framework\DataAbstractionLayer\EntityDefinition;
use Shopware\Core\Framework\Uuid\Uuid;
use Shopware\Elasticsearch\Framework\AbstractElasticsearchDefinition;
use Shopware\Elasticsearch\Framework\ElasticsearchFieldBuilder;
use Shopware\Elasticsearch\Framework\ElasticsearchFieldMapper;
use Shopware\Elasticsearch\Framework\ElasticsearchIndexingUtils;

class YourElasticsearchDefinition extends AbstractElasticsearchDefinition
{
    /**
     * @internal
     */
    public function __construct(
        private readonly EntityDefinition $definition,
        private readonly CompletionDefinitionEnrichment $completionDefinitionEnrichment,
        private readonly ElasticsearchFieldBuilder $fieldBuilder
    ) {
    }

    public function getMapping(Context $context): array
    {
        $languageFields = $this->fieldBuilder->translated(self::getTextFieldConfig());

        $properties = [
            'id' => self::KEYWORD_FIELD,
            'name' => $languageFields,
            'description' => $languageFields,
        ];

        return [
            '_source' => ['includes' => ['id']],
            'properties' => $properties,
        ];
    }

    public function fetch(array $ids, Context $context): array
    {
        $data = $this->fetchData($ids, $context);

        $documents = [];

        foreach ($data as $id => $item) {
            $translations = ElasticsearchIndexingUtils::parseJson($item, 'translation');

            $documents[$id] = [
                'id' => $id,
                'name' => ElasticsearchFieldMapper::translated('name', $translations),
                'description' => ElasticsearchFieldMapper::translated('description', $translations),
            ];
        }

        return $documents;
    }
}
```
## \Shopware\Core\Framework\Log\LoggerFactory:
`\Shopware\Core\Framework\Log\LoggerFactory` will be removed. You can use monolog configuration to achieve the same results. See https://symfony.com/doc/current/logging/channels_handlers.html.
## Removal of separate Elasticsearch exception classes
Removed the following exception classes:
* `\Shopware\Elasticsearch\Exception\ElasticsearchIndexingException`
* `\Shopware\Elasticsearch\Exception\NoIndexedDocumentsException`
* `\Shopware\Elasticsearch\Exception\ServerNotAvailableException`
* `\Shopware\Elasticsearch\Exception\UnsupportedElasticsearchDefinitionException`
* `\Shopware\Elasticsearch\Exception\ElasticsearchIndexingException`
Use the exception factory class `\Shopware\Elasticsearch\ElasticsearchException` instead.
## Configure queue workers to consume low_priority queue
Explicitly configure your workers to additionally consume messages from the `low_priority` queue.
Up to 6.6 the `low_priority` queue is automatically added to the workers, even if not specified explicitly.

Before:
```bash
php bin/console messenger:consume async
```

After:
```bash
php bin/console messenger:consume async low_priority
```

## Configure another transport for the "low priority" queue
The transport defaults to use Doctrine. You can use the `MESSENGER_TRANSPORT_LOW_PRIORITY_DSN` environment variable to change it.

Before:
```yaml
parameters:
    messenger.default_transport_name: 'v65'
    env(MESSENGER_TRANSPORT_DSN): 'doctrine://default?auto_setup=false'
```

After:
```yaml
parameters:
    messenger.default_transport_name: 'v65'
    env(MESSENGER_TRANSPORT_DSN): 'doctrine://default?auto_setup=false'
    env(MESSENGER_TRANSPORT_LOW_PRIORITY_DSN): 'doctrine://default?auto_setup=false&queue_name=low_priority'
```

For further details on transports with different priorities, please refer to the Symfony Docs: https://symfony.com/doc/current/messenger.html#prioritized-transports
## `availabilityRuleId` in `\Shopware\Core\Checkout\Shipping\ShippingMethodEntity`:
* Type changed from `string` to be also nullable and will be natively typed to enforce strict data type checking.
## `getAvailabilityRuleId` in `\Shopware\Core\Checkout\Shipping\ShippingMethodEntity`:
* Return type is nullable.
## `getAvailabilityRuleUuid` in `\Shopware\Core\Framework\App\Lifecycle\Persister\ShippingMethodPersister`:
* Has been removed without replacement.
## `Required` flag for `availability_rule_id` in `\Shopware\Core\Checkout\Shipping\ShippingMethodDefinition`:
* Has been removed.

## ES Definition's buildTermQuery could return BuilderInterface:
* In 6.5 we only allow return `BoolQuery` from `AbstractElasticsearchDefinition::buildTermQuery` method which is not always the case. From next major version, we will allow return `BuilderInterface` from this method.

## Removal of Product Export exception
* Removed `\Shopware\Core\Content\ProductExport\Exception\EmptyExportException` use `\Shopware\Core\Content\ProductExport\ProductExportException::productExportNotFound` instead

## Introduced in 6.5.6.0
## Removal of CacheInvalidatorStorage

The delayed cache invalidation storage was until 6.6 the cache implementation.
As this is not ideal for multi-server usage, we deprecated it in 6.5 and removed it now.
Delaying of cache invalidations now requires a Redis instance to be configured.

```yaml
shopware:
    cache:
        invalidation:
            delay_options:
                storage: cache
                dsn: 'redis://localhost'
```

## Introduced in 6.5.5.0
## New stock handling implementation is now the default

The `product.stock` field is now the primary source for real time product stock values. However, `product.availableStock` is a direct mirror of `product.stock` and is updated whenever `product.stock` is updated via the DAL.

A database migration `\Shopware\Core\Migration\V6_6\Migration1691662140MigrateAvailableStock` takes care of copying the `available_stock` field to the `stock` field.

## New configuration values

* `stock.enable_stock_management` - Default `true`. This can be used to completely disable Shopware's stock handling. If disabled, product stock will be not be updated as orders are created and transitioned through the various states.

## Removed `\Shopware\Core\Content\Product\DataAbstractionLayer\StockUpdater`

The listener was replaced with a new listener `\Shopware\Core\Content\Product\Stock\OrderStockSubscriber` which handles subscribing to the various order events and interfaces with the stock storage `\Shopware\Core\Content\Product\Stock\AbstractStockStorage` to write the stock alterations.

## Removed `\Shopware\Core\Content\Product\SalesChannel\Detail\AbstractAvailableCombinationLoader::load()` && `\Shopware\Core\Content\Product\SalesChannel\Detail\AvailableCombinationLoader::load()`

These methods are removed and superseded by `loadCombinations` which has a different method signature.

From:

```php
public function load(string $productId, Context $context, string $salesChannelId)
```

To:

```php
public function loadCombinations(string $productId, SalesChannelContext $salesChannelContext): AvailableCombinationResult
```

The `loadCombinations` method has been made abstract so it must be implemented. The `SalesChannelContext` instance, contains the data which was previously in the defined on the `load` method. 

`$salesChannelId` can be replaced with `$salesChannelContext->getSalesChannel()->getId()`.

`$context` can be replaced with `$salesChannelContext->getContext()`.

## Writing to `product.availableStock` field is now not possible

The field is write protected. Use the `product.stock` to write new stock levels. 

## Reading product stock

The `product.stock` should be used to read the current stock level. When building new extensions which need to query the stock of a product, use this field. Not the `product.availableStock` field.

## Removed `\Shopware\Core\Framework\DataAbstractionLayer\Event\BeforeDeleteEvent`

It is replaced by `\Shopware\Core\Framework\DataAbstractionLayer\Event\EntityDeleteEvent` with the same API.

You should use `\Shopware\Core\Framework\DataAbstractionLayer\Event\EntityDeleteEvent` instead, only the class name changed.
## sw-field deprecation:
* Instead of `<sw-field type="url"` use `<sw-url-field`. You can see the component mapping in the `sw-field/index.js`

## Introduced in 6.5.4.0
## Removal of `ProductLineItemFactory`
Removed `\Shopware\Core\Content\Product\Cart\ProductLineItemFactory`, use `\Shopware\Core\Checkout\Cart\LineItemFactoryHandler\ProductLineItemFactory` instead.

## Introduced in 6.5.3.0
## Removal of `flow-action-1.0.xsd`
We removed `Shopware\Core\Framework\App\FlowAction\Schema\flow-action-1.0.xsd`, use `Shopware\Core\Framework\App\Flow\Schema\flow-1.0.xsd` instead.
Also use the `Resources/flow.xml` file path instead of `Resources/flow-action.xml` for your apps flow configuration.
## Removal of `Shopware\Core\Framework\App\FlowAction` and `Shopware\Core\Framework\App\FlowAction\Xml`
We moved all class from namespaces `Shopware\Core\Framework\App\FlowAction` to `Shopware\Core\Framework\App\Flow\Action` and `Shopware\Core\Framework\App\FlowAction\Xml` to `Shopware\Core\Framework\App\Flow\Action\Xml`.
Please use new namespaces.
* Removed `\Shopware\Core\Content\Product\SalesChannel\Listing\ProductListingFeaturesSubscriber`, use `CompositeProcessor` instead
## Removal of API-Conversion mechanism

The API-Conversion mechanism was not used anymore, therefore, the following classes were removed:
* `\Shopware\Core\Framework\Api\Converter\ApiVersionConverter`
* `\Shopware\Core\Framework\Api\Converter\ConverterRegistry`
* `\Shopware\Core\Framework\Api\Converter\Exceptions\ApiConversionException`

## Removal of separate Product Export exception classes
Removed the following exception classes:
* `\Shopware\Core\Content\ProductExport\Exception\RenderFooterException`
* `\Shopware\Core\Content\ProductExport\Exception\RenderHeaderException`
* `\Shopware\Core\Content\ProductExport\Exception\RenderProductException`

## Introduced in 6.5.1.0
## `writeAccess` field removed in `integrations`

The `writeAccess` field was removed from the `integration` entity without replacement as it was unused.
## `defaultRunInterval` field is required for `ScheduledTask` entities

The `defaultRunInterval` field is now required for `ScheduledTask` entities. So you now have to provide the following required fields to create a new Scheduled Task in the DB:
* `name`
* `scheduledTaskClass`
* `runInterval`
* `defaultRunInterval`
* `status`
## Removed `\Shopware\Core\Content\Media\DeleteNotUsedMediaService`
All usages of `\Shopware\Core\Content\Media\DeleteNotUsedMediaService` should be replaced with `\Shopware\Core\Content\Media\UnusedMediaPurger`. There is no replacement for the `countNotUsedMedia` method because counting the number of unused media on a system with a lot of media is time intensive.
The `deleteNotUsedMedia` method exists on the new service but has a different signature. `Context` is no longer required. To delete only entities of a certain type it was previously necessary to add an extension to the `Context` object. Instead, pass the entity name to the third parameter of `deleteNotUsedMedia`.
The first two parameters allow to process a slice of media, passing null to those parameters instructs the method to check all media, in batches.
* Changed the following classes to be internal:
  - `\Shopware\Core\Framework\Webhook\Hookable\HookableBusinessEvent`
  - `\Shopware\Core\Framework\Webhook\Hookable\HookableEntityWrittenEvent`
  - `\Shopware\Core\Framework\Webhook\Hookable\HookableEventFactory`
  - `\Shopware\Core\Framework\Webhook\Hookable\WriteResultMerger`
  - `\Shopware\Core\Framework\Webhook\Message\WebhookEventMessage`
  - `\Shopware\Core\Framework\Webhook\ScheduledTask\CleanupWebhookEventLogTask`
  - `\Shopware\Core\Framework\Webhook\BusinessEventEncoder`
  - `\Shopware\Core\Framework\Webhook\WebhookDispatcher`
## FlowEventAware interface change 
With v6.6 we change the class hierarchy of the following flow event interfaces:
* `CustomerRecoveryAware`
* `MessageAware`
* `NewsletterRecipientAware`
* `OrderTransactionAware`
* `CustomerAware`
* `CustomerGroupAware`
* `MailAware`
* `OrderAware`
* `ProductAware`
* `SalesChannelAware`
* `UserAware`
* `LogAware`

When you have implemented one of these interfaces in one of your own event classes, you should now also implement the `FlowEventAware` interface by yourself.
This is necessary to ensure that your event class is compatible with the new flow event system.

**Before:**
```php
<?php declare(strict_types=1);

namespace App\Event;

use Shopware\Core\Framework\Log\LogAware;

class MyEvent implements LogAware
{
    // ...
}
```

**After:**

```php
<?php declare(strict_types=1);

namespace App\Event;

use Shopware\Core\Framework\Event\FlowEventAware;

class MyEvent implements FlowEventAware, LogAware
{
    // ...
}
```
## Indexer Offset Changes

The methods `setNextLanguage()` and `setNextDefinition()` in `\Shopware\Elasticsearch\Framework\Indexing\IndexerOffset` are removed, use `selectNextLanguage()` or `selectNextDefinition()` instead.
Before:
```php 
$offset->setNextLanguage($languageId);
$offset->setNextDefinition($definition);
```

After:
```php
$offset->selectNextLanguage($languageId);
$offset->selectNextDefinition($definition);
```

## Changes to data-attribute selector names

We want to change several data-attribute selector names to be more aligned with the JavaScript plugin name which is initialized on the data-attribute selector.
When you use one of the selectors listed below inside HTML/Twig, JavaScript or CSS, please change the selector to the new selector.

## HTML/Twig example

### Before

```twig
<div 
    data-offcanvas-menu="true" {# <<< Did not match options attr #}
    data-off-canvas-menu-options='{ ... }'
>
</div>
```

### After

```twig
<div 
    data-off-canvas-menu="true" {# <<< Now matches options attr #}
    data-off-canvas-menu-options='{ ... }'
>
</div>
```

_The options attribute is automatically generated using the camelCase JavaScript plugin name._

## Full list of selectors

| old                             | new                              |
|:--------------------------------|:---------------------------------|
| `data-search-form`              | `data-search-widget`             |
| `data-offcanvas-cart`           | `data-off-canvas-cart`           |
| `data-collapse-footer`          | `data-collapse-footer-columns`   |
| `data-offcanvas-menu`           | `data-off-canvas-menu`           |
| `data-offcanvas-account-menu`   | `data-account-menu`              |
| `data-offcanvas-tabs`           | `data-off-canvas-tabs`           |
| `data-offcanvas-filter`         | `data-off-canvas-filter`         |
| `data-offcanvas-filter-content` | `data-off-canvas-filter-content` |

## Introduced in 6.5.0.0
## Removed `SyncOperationResult`
The `\Shopware\Core\Framework\Api\Sync\SyncOperationResult` class was removed without replacement, as it was unused.
## Removal of `MessageSubscriberInterface` for `ScheduledTaskHandler`
The method `getHandledMessages()` in abstract class `\Shopware\Core\Framework\MessageQueue\ScheduledTask\ScheduledTaskHandler` was removed, please use the `#[AsMessageHandler]` attribute instead.

Before:
```php
class MyScheduledTaskHandler extends ScheduledTaskHandler
{
    public static function getHandledMessages(): iterable
    {
        return [MyMessage::class];
    }
    
    public function run(): void
    {
        // ...
    }
}
```

After: 
```php
#[AsMessageHandler(handles: MyMessage::class)]
class MyScheduledTaskHandler extends ScheduledTaskHandler
{
    public function run(): void
    {
        // ...
    }
}
```
## Deprecated component `sw-dashboard-external-link` has been removed
* Use component `sw-external-link` instead of `sw-dashboard-external-link`
## Selector to open an ajax modal
The selector to initialize the `AjaxModal` plugin will be changed to not interfere with Bootstrap defaults data-attribute API.

### Before
```html
<a data-bs-toggle="modal" data-url="/my/route" href="/my/route">Open Ajax Modal</a>
```

### After
```html
<a data-ajax-modal="true" data-url="/my/route" href="/my/route">Open Ajax Modal</a>
```
## `IsNewCustomerRule` to be removed with major release v6.6.0
* Use `DaysSinceFirstLoginRule` instead with operator `=` and `daysPassed` of `0` to achieve identical behavior
## Seeding mechanism for `AbstractThemePathBuilder`

The `generateNewPath()` and `saveSeed()` methods  in `\Shopware\Storefront\Theme\AbstractThemePathBuilder` are now abstract, this means you should implement those methods to allow atomic theme compilations.

For more details refer to the corresponding [ADR](../../adr/storefront/2023-01-10-atomic-theme-compilation.md).

## Removal of `blacklistIds` and `whitelistIds` in  `\Shopware\Core\Content\Product\ProductEntity`
Two properties `blacklistIds` and `whitelistIds` were removed without replacement

## Replace `@shopware-ag/admin-extension-sdk` with `@shopware-ag/meteor-admin-sdk`

### Before
```json
{
    "dependencies": {
        "@shopware-ag/admin-extension-sdk": "^3.0.14"
    }
}
```

### After
```json
{
    "dependencies": {
        "@shopware-ag/meteor-admin-sdk": "^3.0.16"
    }
}
```

## Update `@shopware-ag/meteor-admin-sdk` to `^4.0.0`

### Before
```json
{
    "dependencies": {
        "@shopware-ag/meteor-admin-sdk": "^3.0.17"
    }
}
```

### After
```json
{
    "dependencies": {
        "@shopware-ag/meteor-admin-sdk": "^4.0.0"
    }
}
```

## Administration tooltips no longer support components/ html
Due to a Vue 3 limitation the `v-tooltip` directive no longer supports components or html.

### Before
```html
<div
    v-tooltip="{
        message: 'For more information click <a href=\"https://shopware.com\">here</a>.',
    }"
</div>
```

### After
```html
<div
    v-tooltip="{
        message: 'For more information visit shopware.com',
    }"
</div>
```
