# jscalc

***

### Exploiting JavaScript `eval()` Code Injection

In the mysterious depths of the digital sea, a specialized JavaScript calculator has been crafted. With multiple arms and complex problem-solving skills, it's used for everything from inkjet trajectory calculations to deep-sea math. I decided to attempt to outsmart this calculator—here's how I did it.

### **Understanding the Calculator’s Functionality**

First, explore the website and understand its core functionality. In this case, the website evaluates a mathematical expression and returns the result using the `eval()` function. This function in JavaScript is used to execute arbitrary JavaScript code represented as a string. While it's powerful, it's also potentially dangerous.

#### Example Code:

I had access to the source code and found this piece of code demonstrating how `eval()` is used:

```js
const path       = require('path');
const express    = require('express');
const router     = express.Router();
const Calculator = require('../helpers/calculatorHelper');
const response = data => ({ message: data });
router.get('/', (req, res) => {
    return res.sendFile(path.resolve('views/index.html'));
});
router.post('/api/calculate', (req, res) => {
    let { formula } = req.body;
    if (formula) {
        result = Calculator.calculate(formula);
        return res.send(response(result));
    }
    return res.send(response('Missing parameters'));
})
module.exports = router;

//Calculator.calculate(formula) call calculate function from calculatorHelper declared in the require above.

module.exports = {
    calculate(formula) {
        try {
            return eval(`(function() { return ${ formula } ;}())`);
        } catch (e) {
            if (e instanceof SyntaxError) {
                return 'Something went wrong!';
            }
        }
    }
}
```

#### **Exploitation**

Now, let’s dive into how to exploit this vulnerability. Understanding the underlying system helps us craft a payload to execute commands or retrieve sensitive data.

1. **Initial Exploration**: The application works by evaluating a mathematical expression entered by the user. After inputting various expressions, I found that some commands could be executed in this unfiltered environment.
2. **Crashing the Application**: We start by trying to make the application crash. This can help determine how much control we can gain over the execution.
3.  **Testing for Command Execution**: We can try using JavaScript functions that are often available in the Node.js environment. Examples include:

    * `process.platform`
    * `process.cwd()`&#x20;

    <figure><img src="https://vk9-sec.com/wp-content/uploads/2024/01/word-image-9817-4.png" alt=""><figcaption></figcaption></figure>



    \==**As demonstrated by the source code, there is no sanitization or validation of the input. It is directly parsed within the `calculate` function without any filtering, making it a significant vulnerability that can be exploited. It is strongly recommended to avoid using the `eval()` function, as it is prone to command injection attacks.==** **==Sources: OWASP - Command Injection, OWASP - Eval() Security Risk.**==
4.  **Executing System Commands**: Now that we know we can execute arbitrary commands, I decided to run a more sophisticated command:

    javascript

    CopierModifier

    `require('child_process').execSync('ls -l').toString();`

    Here’s what each part of the command does:

    * **`require('child_process')`**: This imports the `child_process` module in Node.js, which is used to spawn child processes and execute shell commands.
    * **`execSync('ls -l')`**: This runs the `ls -l` command synchronously, which lists files in the current directory in long format.
    * **`.toString()`**: The result of `execSync()` is a `Buffer` object, so `.toString()` converts it into a string.
5. **Exploit Success**: By injecting this command into the calculator's input, we can execute system commands and access potentially sensitive information.
