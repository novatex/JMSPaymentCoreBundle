Plugins
=======

Introduction
------------
A plugin is a flexible way of providing access to a specific payment back end, 
payment processor, or payment service provider.

.. note ::

    If you are coming from symfony1, the term "plugin" as used by this bundle
    has nothing to do with symfony1's plugin extension system.
    
Plugins are used to execute :doc:`financial transactions <model/FinancialTransaction>` 
against a payment service provider, such as Paypal.

Implementing a Custom Plugin
----------------------------
The easiest way is to simply extend the provided ``AbstractPlugin`` class, and override
the remaining abstract methods:

.. code-block :: php

    <?php

    class PaypalPlugin extends \JMS\Payment\CoreBundle\Plugin\AbstractPlugin
    {
        public function processes($name)
        {
            return 'paypal' === $name;
        }
    }
    
Now, you only need to set-up your plugin as a service, and it will be added to the
plugin controller automatically:

.. configuration-block ::

    .. code-block :: yaml
    
        services:
            payment.plugin.paypal:
                class: PaypalPlugin
                tags: [{name: payment.plugin}]

    .. code-block :: xml

        <service id="payment.plugin.paypal" class="PaypalPlugin">
            <tag name="payment.plugin" />
        </service>
    
That's it! You just created your first plugin :) Right now, it does not do anything
useful, but we will get to the specific transactions that you can perform in
the next section. 

Available Transaction Types
---------------------------
Each plugin may implement a variety of available transaction types. Depending on the
used payment method, and the capabilities of the backend, you rarely need all of them.

Following is a list of all available transactions, and two exemplary payment method
plugins. A "x" indicates that the method is implement, "-" that it is not:

+----------------------------+------------------+-----------------------+ 
| Financial Transaction      | CreditCardPlugin | ElectronicCheckPlugin |
+============================+==================+=======================+
| checkPaymentInstruction    |        x         |           x           |
+----------------------------+------------------+-----------------------+
| validatePaymentInstruction |        x         |           x           |
+----------------------------+------------------+-----------------------+
| approveAndDeposit          |        x         |           x           |
+----------------------------+------------------+-----------------------+
| approve                    |        x         |          \-           |
+----------------------------+------------------+-----------------------+
| reverseApproval            |        x         |          \-           |
+----------------------------+------------------+-----------------------+
| deposit                    |        x         |           x           | 
+----------------------------+------------------+-----------------------+
| reverseDeposit             |        x         |          \-           |
+----------------------------+------------------+-----------------------+
| credit                     |        x         |          \-           |
+----------------------------+------------------+-----------------------+
| reverseCredit              |        x         |          \-           |
+----------------------------+------------------+-----------------------+

If you are unsure which transactions to implement, have a look at the ``PluginInterface``
which contains detailed descriptions for each of them.

.. tip ::

    In cases, where a certain method does not make sense for your payment backend,
    you should throw a ``FunctionNotSupportedException``. If you extend the ``AbstractPlugin``
    base class, this is already done for you.
    
Available Exceptions
--------------------
Exceptions play an important part in the communication between the different payment plugin,
and the ``PluginController`` which manages them. 

Following is a list with available exceptions, and how they are treated by the ``PluginController``.
Of course, you can also add your own exceptions, but it is recommend that you sub-class 
an existing exception when doing so. 

.. tip ::

    All exceptions which are relevent for plugins are located in the namespace 
    ``JMS\Payment\CoreBundle\Plugin\Exception``.

+------------------------------------+-----------------------------+---------------------------+
| Class                              | Description                 | Payment Plugin Controller |
|                                    |                             | Interpretation            |
+====================================+=============================+===========================+
| Exception                          | Base exception used by all  | Causes any transaction to |
|                                    | exceptions thrown from      | be rolled back. Exception |
|                                    | plugins.                    | will be re-thrown.        |
+------------------------------------+-----------------------------+---------------------------+
| FunctionNotSupportedException      | This exception is thrown    | In most cases, this causes|
|                                    | whenever a method on the    | any transactions to be    |
|                                    | interface is not supported  | rolled back. Notable      |
|                                    | by the plugin.              | exceptions to this rule:  |
|                                    |                             | checkPaymentInstruction,  |
|                                    |                             | validatePaymentInstruction|
+------------------------------------+-----------------------------+---------------------------+
| InvalidDataException               | This exception is thrown    | Causes any transaction to |
|                                    | whenever the plugin realizes| be rolled back. Exception |
|                                    | that the data associated    | will be re-thrown.        |
|                                    | with the transaction is     |                           |
|                                    | invalid.                    |                           |
+------------------------------------+-----------------------------+---------------------------+
| InvalidPaymentInstructionException | This exception is typically | Causes PaymentInstruction |
|                                    | thrown from within either   | to be set to              |
|                                    | checkPaymentInstruction, or | STATE_INVALID.            |
|                                    | validatePaymentInstruction. |                           |
+------------------------------------+-----------------------------+---------------------------+
| BlockedException                   | This exception is thrown    | Sets the transaction to   |
|                                    | whenever a transaction      | STATE_PENDING, and        |
|                                    | cannot be processed.        | converts the exception to |
|                                    |                             | a Result object.          |
|                                    | The exception must only be  |                           |
|                                    | used when the situation is  |                           |
|                                    | temporary, and there is a   |                           |
|                                    | chance that the transaction |                           |
|                                    | can be performed at a later |                           |
|                                    | time successfully.          |                           |
+------------------------------------+-----------------------------+---------------------------+
| TimeoutException                   | This exception is thrown    | Sets the transaction to   |
| (sub-class of BlockedException)    | when there is an enduring   | STATE_PENDING, and        |
|                                    | communicaton problem with   | converts the exception to |
|                                    | the payment backend system. | a Result object.          |
+------------------------------------+-----------------------------+---------------------------+
| ActionRequiredException            | This exception is thrown    | Sets the transaction to   |
| (sub-class of BlockedException)    | whenever an action is       | STATE_PENDING, and        |
|                                    | required before the         | converts the exception to |
|                                    | transaction can be completed| a Result object.          |
|                                    | successfully.               |                           |
|                                    |                             |                           |
|                                    | A typical action would be   |                           |
|                                    | for the user to visit an    |                           |
|                                    | URL in order to authorize   |                           |
|                                    | the payment.                |                           |
+------------------------------------+-----------------------------+---------------------------+

Payment-related User Data
-------------------------
The Form Type
~~~~~~~~~~~~~
The form type is necessary for collecting, and validating the user data that is necessary 
for your payment method. In the following, we assume that we are designing a form type for
credit card payment which could look like this:

.. code-block :: php

    <?php
    
    use Symfony\Component\Form\AbstractType;
    use Symfony\Component\Form\FormBuilder;
    
    class CreditCardType extends AbstractType
    {
        public function buildForm(FormBuilder $builder, array $options)
        {
            $builder
                ->add('holder', 'text', array('required' => false))
                ->add('number', 'text', array('required' => false))
                ->add('expires', 'date', array('required' => false))
                ->add('code', 'text', array('required' => false))
            ;
        }
    
        public function getName()
        {
            return 'credit_card';
        }
    }

.. note ::

    Make sure to declare all fields as non-required. This is merely affecting
    the client-side validation, server-side validation is not affected.

Configuring Your Form Type
~~~~~~~~~~~~~~~~~~~~~~~~~~
Now, we need to wire the form type with the dependency injection container:

.. configuration-block ::

    .. code-block :: yaml
    
        services:
            credit_card_type:
                class: CreditCardType
                tags:
                    - { name: form.type, alias: credit_card }
                    - { name: payment.method_type }
                    
    .. code-block :: xml
     
        <service id="credit_card_type" class="CreditCardType">
            <tag name="form.type" alias="credit_card" />
            <tag name="payment.method_type" />
        </service>    

Validating the Submitted User Data
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Validation is handled by your ``Plugin`` class. It contains two methods for this:

#. ``checkPaymentInstruction`` (fast): validates the submitted data, but does not make any API calls to an external service
#. ``validatePaymentInstruction`` (thorough): does everything that ``checkPaymentInstruction`` does, but may also make API calls

We are now going to implement the ``checkPaymentInstruction`` method for our form type above:

.. code-block :: php

    <?php
    
    use JMS\Payment\CoreBundle\Plugin\AbstractPlugin;
    use JMS\Payment\CoreBundle\Model\PaymentInstructionInterface;
    use JMS\Payment\CoreBundle\Plugin\ErrorBuilder;
    
    class CreditCardPlugin extends AbstractPlugin
    {
        public function checkPaymentInstruction(PaymentInstructionInterface $instruction)
        {
            $errorBuilder = new ErrorBuilder();
            $data = $instruction->getExtendedData();
            
            if (!$data->get('holder')) {
                $errorBuilder->addDataError('holder', 'form.error.required');
            }
            if (!$data->get('number')) {
                $errorBuilder->addDataError('number', 'form.error.required');
            }
            
            if ($instruction->getAmount() > 10000) {
                $errorBuilder->addGlobalError('form.error.credit_card_max_limit_exceeded');
            }
            
            // more checks here ...
            
            if ($errorBuilder->hasErrors()) {
                throw $errorBuilder->getException();
            }
        }
    
        public function processes($method)
        {
            return 'credit_card' === $method;
        }
    }

.. note ::

    The data errors are automatically mapped to the respective fields of the form.
    Global errors are applied to the form itself.

