## Shopware creating CLI command to generate demo data on your entity

In this blog post, you will learn how to create a custom command in a Shopware 6 plugin and use it to generate the demo data.

Because Shopware 6 is based on the Symfony framework, it also has its Console available. So you can do most of the stuff you need to do while developing Shopware 6 plugins.

## Folder structure

```jsx
<pluginRoot>
└── src
    ├── Command
    │   └── DemodataCommand.php
    │
    ├── Generator
    │   └── ArticleGenerator.php
    │
    └── Resources
        └── config
            └── services.xml
```

## Registering the command
To register a new command, just add it to your plugin's `services.xml` and specify the `console.command` tag:

```xml
<service id="Sas\BlogModule\Command\DemodataCommand">
    <argument type="service" id="Shopware\Core\Framework\Demodata\DemodataService"/>
    <tag name="console.command"/>
</service>
```

## Configuring the command
Your command's class should extend from the `Symfony\Component\Console\Command\Command`
 class:

```php
class DemodataCommand extends Command
{
    protected static $defaultName = '';

    protected function configure(): void
    {
        //
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        //
    }
}
```

Let’s start with the name of the command (the part after "bin/console"):

```php
// the command description shown when running "php bin/console list"
protected static $defaultName = 'article:demodata';
```

You can optionally define a description, help message and the [input options and arguments](https://symfony.com/doc/current/console/input.html)
 by overriding the `configure()` method:

```php
protected function configure(): void
{
    $this->addArgument('count', InputArgument::REQUIRED, 'The number of the articles.');
}
```

Put in the `execute()` method the code to create demo data:
This method must return an integer number with the "exit status code" of the command. You can also use these constants to make code more readable:

- `Command::SUCCESS` if there was no problem running the command.
- `Command::FAILURE` if some error happened during the execution.
- `Command::INVALID` indicates incorrect command usage: invalid options or missing arguments.

```php
protected function execute(InputInterface $input, OutputInterface $output): int
{
    // ... put here the code to create articles

    return Command::SUCCESS;
}
```

## Styling the command

**Title**

It displays the given string as the command title. This method is meant to be used only once in a given command, but nothing prevents you from using it repeatedly:

```php
$io->title('Article Data Generator');
```

The console output should be:

```bash
Article Data Generator
======================
```

**Table**

It displays the given array of headers and rows as a compact table:

```php
$io->table(
    ['Entity', 'Items', 'Time'],
    $demoContext->getTimings()
);
```

The console output should be:

```bash
---------------------- ------- --------------------
  Entity                 Items   Time
---------------------- ------- --------------------
  media                  1000    3.7
  article                1000    6.1
---------------------- ------- --------------------
```

**Progress bar**

When executing longer-running commands, it may be helpful to show progress information, which updates as your command runs:

```bash
Generating 1000 items for article
---------------------------------

1000/1000 [============================] 100%

! [NOTE] Took 6.1 seconds
```

Start progress bar: It displays a progress bar with several steps equal to the argument passed to the method (don't progress bar's length of the progress bar is unknown).

```php
// displays a progress bar of unknown length
$context->getConsole()->progressStart();

// displays a 100-step length progress bar
$context->getConsole()->progressStart(100);
```

Advance progress bar: It makes the progress bar advance the given number of steps.

```php
// advances the progress bar 1 step
$context->getConsole()->progressAdvance();

// advances the progress bar 10 steps
$context->getConsole()->progressAdvance(10);
```

Finish progress bar: It finishes the progress bar (filling up all the remaining steps when its length is known).

```php
$context->getConsole()->progressFinish();
```

## Creating the generator

To register a new generator, add it to your plugin's `services.xml` and specify the `shopware.demodata_generator` tag:
```xml
<service id="Sas\BlogModule\Generator\ArticleGenerator">
    <argument type="service" id="Shopware\Core\Framework\DataAbstractionLayer\Write\EntityWriter" />
    <argument type="service" id="Doctrine\DBAL\Connection" />
    <argument type="service" id="Sas\BlogModule\Content\Article\ArticleDefinition"/>
    <tag name="shopware.demodata_generator"/>
</service>
```

Now, we need our generator to know its definition class. This is done by overriding the method `getDefinition()` 
```php
public function getDefinition(): string
{
    return ArticleDefinition::class;
}
```

Put in the `generate()` method the code to generate demo data:
```php
public function generate(int $numberOfItems, DemodataContext $context, array $options = []): void
{
    $writeContext = WriteContext::createFromContext($context->getContext());

    $payload = [
        // an array of articles
    ];

    $this->writer->upsert($this->articleDefinition, $payload, $writeContext);
}
```

## Running the command

```bash
bin/console article:demodata 1000
```

---

## Complete classes and XML file

**DemodataCommand.php**

```php
use Sas\BlogModule\Content\Article\ArticleDefinition;
use Shopware\Core\Framework\Adapter\Console\ShopwareStyle;
use Shopware\Core\Framework\Context;
use Shopware\Core\Framework\Demodata\DemodataRequest;
use Shopware\Core\Framework\Demodata\DemodataService;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputArgument;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

class DemodataCommand extends Command
{
    protected static $defaultName = 'article:demodata';

    private DemodataService $demodataService;

    public function __construct(DemodataService $demodataService)
    {
        parent::__construct();

        $this->demodataService = $demodataService;
    }

    protected function configure(): void
    {
        $this->addArgument('count', InputArgument::REQUIRED, 'The number of the articles.');
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $io = new ShopwareStyle($input, $output);
        $io->title('Article Data Generator');

        $context = Context::createDefaultContext();

        $request = new DemodataRequest();

        $request->add(ArticleDefinition::class, (int)$input->getArgument('count'));

        $demoContext = $this->demodataService->generate($request, $context, $io);

        $io->table(
            ['Entity', 'Items', 'Time'],
            $demoContext->getTimings()
        );

        return self::SUCCESS;
    }
}
```

**ArticleGenerator.php**

```php
use Doctrine\DBAL\Connection;
use Sas\BlogModule\Content\Article\ArticleDefinition;
use Shopware\Core\Framework\DataAbstractionLayer\Write\EntityWriterInterface;
use Shopware\Core\Framework\DataAbstractionLayer\Write\WriteContext;
use Shopware\Core\Framework\Demodata\DemodataContext;
use Shopware\Core\Framework\Demodata\DemodataGeneratorInterface;
use Shopware\Core\Framework\Uuid\Uuid;

class ArticleGenerator implements DemodataGeneratorInterface
{
    private EntityWriterInterface $writer;

    private Connection $connection;

    private ArticleDefinition $articleDefinition;

    public function __construct(
        EntityWriterInterface $writer,
        Connection            $connection,
        ArticleDefinition     $articleDefinition
    ) {
        $this->writer = $writer;
        $this->connection = $connection;
        $this->articleDefinition = $articleDefinition;
    }

    public function getDefinition(): string
    {
        return ArticleDefinition::class;
    }

    public function generate(int $numberOfItems, DemodataContext $context, array $options = []): void
    {
        $authorId = $this->connection->fetchOne('SELECT LOWER(HEX(id)) FROM author');

        $writeContext = WriteContext::createFromContext($context->getContext());

        $context->getConsole()->progressStart($numberOfItems);

        $payload = [];
        for ($i = 0; $i < $numberOfItems; ++$i) {
						$translations = [
                'en-GB' => [
                    'title' => $title,
                    'teaser' => $context->getFaker()->text(200),
                    'content' => $context->getFaker()->text(50),
                ],
            ];

            $article = [
                'id' => Uuid::randomHex(),
                'active' => true,
                'authorId' => $authorId,
                'translations' => $translations
            ];

            $payload[] = $article;

            if (\count($payload) >= 100) {
                $this->writer->upsert($this->articleDefinition, $payload, $writeContext);
                $context->getConsole()->progressAdvance(\count($payload));
                $payload = [];
            }
        }

        if (!empty($payload)) {
            $this->writer->upsert($this->articleDefinition, $payload, $writeContext);
            $context->getConsole()->progressAdvance(\count($payload));
        }

        $context->getConsole()->progressFinish();
    }
}
```

**services.xml**

```xml
<?xml version="1.0" ?>

<container xmlns="http://symfony.com/schema/dic/services"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="http://symfony.com/schema/dic/services http://symfony.com/schema/dic/services/services-1.0.xsd">

    <services>

        <service id="Sas\BlogModule\Command\DemodataCommand">
            <argument type="service" id="Shopware\Core\Framework\Demodata\DemodataService"/>
            <tag name="console.command"/>
        </service>

        <service id="Sas\BlogModule\Generator\ArticleGenerator">
            <argument type="service" id="Shopware\Core\Framework\DataAbstractionLayer\Write\EntityWriter" />
            <argument type="service" id="Doctrine\DBAL\Connection" />
            <argument type="service" id="Sas\BlogModule\Content\Article\ArticleDefinition"/>
            <tag name="shopware.demodata_generator"/>
        </service>

    </services>
</container>
```

---

*I am really happy to receive your feedback on this article. Thanks for your precious time reading this.*

**References:**

- [https://developer.shopware.com/docs/guides/plugins/plugins/plugin-fundamentals/add-custom-commands](https://developer.shopware.com/docs/guides/plugins/plugins/plugin-fundamentals/add-custom-commands)
- [https://symfony.com/doc/current/console.html](https://symfony.com/doc/current/console.html)