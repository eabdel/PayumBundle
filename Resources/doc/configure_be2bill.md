## Configure paypal express checkout payment

### Step 1. Download paypal payum lib

Add the following lines in your `composer.json` file:

```json
{
    "require": {
        "payum/be2bill": "dev-master"
    }
}
```

**Note:** You may want to adapt this line to use a specific version.

Now, run composer.phar to download the bundle:

```bash
$ php composer.phar install
```

**Note:** You can immediately start using it. The autoloading files have been generated by composer and already included to the app autoload file.

### Step 2: Configure payum bundle

```yaml

payum:
    contexts:
        simple_sell_be2bill:
            be2bill_payment:
                api:
                    options:
                        identifier: ''
                        password: ''
                        sandbox: true
            filesystem_storage:
                model_class: Payum\Domain\SimpleSell
                storage_dir: %kernel.root_dir%\Resources\payments
                id_property: id
```

Lets describe some parts of the config. 
Payum bundle uses contexts to handle payments. 
You can have as many context as you want.
It is up to you how to name the content. 
We choose filesystem as storage, you may want to use doctrine storage.
You should take api options from be2bill side and put them to `api.options` section. 
Also you should not forget to set `sandbox` option to false in production use. 

**Note:** Dont forget to configure api options properly. 

### Step 3: Configure create instruction action

First we have to create payum action. This action create payment specific instruction from your model. 

```php
<?php
namespace AcmeDemoBundle\Payum\Action;

use Symfony\Component\DependencyInjection\ContainerInterface;

use Payum\Action\ActionPaymentAware;
use Payum\Request\CaptureRequest;
use Payum\Domain\InstructionAggregateInterface;
use Payum\Domain\InstructionAwareInterface;
use Payum\Exception\RequestNotSupportedException;
use Payum\Be2Bill\PaymentInstruction;

class CaptureSimpleSellWithBe2BillAction extends ActionPaymentAware 
{
    protected $container;
    
    public function __construct(ContainerInterface $container)
    {
        $this->container = $container;
    }
    
    /**
     * {@inheritdoc}
     */
    public function execute($request)
    {
        /** @var $request CaptureRequest */
        if (false == $this->supports($request)) {
            throw RequestNotSupportedException::createActionNotSupported($this, $request);
        }

        $simpleSell = $request->getModel();
        
        //be2bill amount format is cents: for example:  100.05 (EUR). will be 10005.
        $instruction = new PaymentInstruction;
        $instruction->setAmount((float) number_format($simpleSell->getPrice(), 2) * 100);
        $instruction->setClientemail('user@email.com');
        $instruction->setClientuseragent($this->container->get('request')->headers->get('User-Agent', 'Unknown'));
        $instruction->setClientip($this->container->get('request')->getClientIp());
        $instruction->setClientident('payerId');
        $instruction->setDescription('Payment for digital stuff');
        $instruction->setOrderid('orderId');
        $instruction->setCardcode($this->container->get('request')->get('cardNumber'));
        $instruction->setCardcvv($this->container->get('request')->get('cardCvv'));
        $instruction->setCardfullname($this->container->get('request')->get('cardFullname'));
        $instruction->setCardvaliditydate($this->container->get('request')->get('cardValiditydate'));
        
        $this->payment->execute(new CaptureRequest($instruction));
    }

    /**
     * {@inheritdoc}
     */
    public function supports($request)
    {
        return
            $request instanceof CaptureRequest &&
            $request->getModel() instanceof SimpleSell
        ;
    }
}
```

Second we have to add this service to container.

```yaml
services:
    payum.action.capture_simple_sell_with_be2bill:
        class:                                                    AcmeDemoBundle\Payum\Action\CaptureSimpleSellWithBe2BillAction
        arguments:
            -                                                     @service_container
            
payum:
    context:
        your_context:
            be2bill_payment:
                #..
                actions:
                    - payum.action.capture_simple_sell_with_be2bill
```

### Next Step

Now you are ready to read [how to capture payment](capture_payment.md).