Visit this link for the full reverse of Botguard: [cdn.csolver.net/botguard](https://cdn.csolver.net/botguard)

This project belong to Cypa.


# ReCaptcha BotGuard

Before we begin the process of reversing BotGuard, let's start with some information.

BotGuard is one of the most sophisticated antibots; it is known as the most difficult antibot to reverse.
It is used on many Google products, but each script serves a different purpose.
The one I present in this repository is ReCaptcha's BotGuard; they use it on their primary payload (`/reload`), and in ReCaptcha v2, they use it on the submit endpoint as well (`/userverify`)

Now that you know a little about where it is used, let's understand what exactly it is.
BotGuard is a script that generates a token, which is used to verify the validity of requests to Google's servers. In ReCaptcha, it is not used as a fingerprint, which many believe it is; instead, it is used to verify that the request is from a browser, as it can only be generated if executed by a browser.
While this may seem like a "weak" approach, it is actually very sophisticated; there are many checks against falsey environments, so patching and sandboxing are no longer viable options. If you do not wish to reverse it but still want to generate it, you will need a browser emulator that can bypass its detection. Tools such as Puppeteer or Playwright can be helpful for this purpose. They emulate a real browser so that the environment checks won’t be effective against them. Due to this, you won’t need to do much setup to create a valid token generator as long as it’s a real browser.

# What Is a VM?

Before we delve into the fun stuff, we need to understand what a VM is and how we can reverse what we don't know. Let's talk about what it is and how it operates. A VM is designed to emulate a CPU running a binary program, but it's custom, written in JavaScript, and the opcodes are often not standard; they usually include some custom ones that aren't available on a CPU. Now, in the case of VMs, it uses this custom "CPU" to execute a "binary" that is the given bytecode. It then decodes or decrypts it, depending on the VM, and begins to implement it. This is one of the most powerful obfuscation techniques at this time; it beats any standard CFF (control flow flattening) by a lot, since you need to write a whole disassembler and decompiler to understand what it does, then, after that, you still have to find and reverse the algorithm that was compiled in the first place.

With VMs, there are multiple types; we have register-based, which emulate that of a modern CPU. The browser you're using? That binary is being run through your register-based CPU. Stack-based is based on older CPUs and JVMs (Java VMs) use stack-based; WASM (WebAssembly) uses stack-based. That should help you understand what a VM is and how it works. Now that we've got that out of the way, we can continue reversing BotGuard.

# What Makes It Difficult?

Many people have tried and failed to reverse this antibot, and because of this, it has a reputation for being the "hardest" antibot, but what exactly causes such trouble?
The script isn't like any other script, where you can debug a little, understand the algorithm, and translate it to a production language, and you're done.
This script utilizes a VM (Virtual Machine), which completely virtualizes the code, making it impossible to locate the algorithm or debug into it, unlike many other scripts.
You need to write a **disassembler** and **decompiler** for this custom bytecode interpreter, which makes it even harder. Even then, if you have experience with VMs, this shouldn't be so hard. **WRONG!** This VM has much more to it than standard ones you see in other VMs like **Kasada** or **TikTok**. They utilize numerous special features, including register encryptions and flow-changing opcodes.

So what? It has more features, but we can still debug to figure it out! Well, they've got some **anti-debug** measures which make it very hard to debug without the script throwing you onto a random path with incorrect flow.
The most obvious check they have is timing; they use `performance.now()` along with `+Date.now()` to check the time. This allows the script to determine if a breakpoint has been placed; if the time is too high, the user was likely broken previously. In the screenshot below, you can see how it changes the **seed**, which is used to determine the next byte, so it diverts the program when debugging is detected. 

# Anti-Debugging Methods

Now, the `K` variable is the VM context; it holds critical globals the VM uses, including the **seed**, which in this case is `K.U`. Notice how it has some quick math operations to determine an amount to xor the seed by? This should remain at `0`, and if it isn't, the value changes; this also causes the following bytes in the program to change, making it a very effective anti-debug method.

<img src="images/photo_1.jpg" alt="image" width="100%">

Okay, so we haven't even started reversing, and it's already looking crazy. We've just covered one of the most obvious anti-debug methods they use, but another interesting one I'd like to cover before we continue is the `anti-logger`. Unlike the anti-debugger, the anti-logger will throw it off course if it detects logging, which includes `logpoints` and calling `console.log` to inspect variables while paused. Below is another screenshot of how they override properties and bind them to change VM globals (the `K` variable)
 
- ⚠️ (I replaced `f[L]` with `f.create` due to `L` being static, I apologize if this makes it harder to follow) ⚠️

<img src="images/photo_2.jpg" alt="image" width="100%">

Now, this doesn't look too bad at first glance, but upon closer examination, we can discover what exactly it does. This function takes **two arguments**: `L` and `N`.
`L` is static, always a string `"create"`, and `N` is the function that sets all the properties to override, which sets up the `anti-logger`.
Let's take a step back and examine what the function `N` actually does, since we know it serves as the handler for the `anti-logger`. Therefore, we should determine its purpose to throw off the program.
If we go back to the caller of `Rr`, we find this block of code. This tells us precisely what happens now.

<img src="images/photo_3.jpg" alt="image" width="100%">

This indicates that it sends a getter method to the `Rr` function, meaning that whenever any of the patched functions are called, it executes the code inside the `get` function, which we know is just `Cu`.
Let's take a closer look at the `Cu` function, as it always returns something whenever it is called. Let's take a look at the origin of the `Qr` variable, since without prior knowledge of how this sort of stuff works, it seems like it just pops a value off `Qr`, but that's not the case, as we see here, `Qr` is an empty array.

<img src="images/photo_4.jpg" alt="image" width="100%">

This means we are getting the `pop` method and binding it to `t.prototype[vp]`, which effectively means that whenever we call any of the previously patched methods on an object, it pops from the `t.prototype[vp]` stack.
This stack is used again in the memory reader function, which retrieves bytes from memory. Therefore, if this value changes, the bytes collected from memory will also change, causing the program to divert again.

# Reversing The VM

Alright, so now we know how they prevent us from debugging. We now know that we cannot set breakpoints or log values at runtime, as it causes the program to break. This definitely has a lot more problems to deal with when reversing compared to a standard VM.

Let's begin by looking for the main point in the VM. We can find that in the `new window.botguard.bg()` call it initializes the VM, this is precisely what we are looking for. Could you take a look at this function we've just found? Modifying a large string (the VM bytecode) before sending it to another function.

<img src="images/photo_5.jpg" alt="image" width="100%">

We know `L` is the bytecode, so let's see what it's doing with it. I see this: `L.substring(0, 3) + "_"` and I want to know what `F[L.substring(0, 3) + "_"]` is, it seems to be the function that initializes the VM. This is already unique, as it utilizes part of the given bytecode to call the VM, which I haven't seen before, but it's exciting to observe. Let's see what `F.LbE_` is since `"LbE_"` is what we get from `L.substring(0, 3) + "_"`.
`F.LbE_` is the initialization function. We can see that it returns an array of two functions, which are later used to set up the `invoke` function, which generates the final token. 

<img src="images/photo_6.jpg" alt="image" width="100%">

Suppose we check out the `invoke` function. In that case, we can figure out that the first function in the returned array from the `$i` function is what generates the token, taking the newly created VM context and a currently undefined variable as arguments. If we follow this a little further, we find a call that leads to a function that appears to decode the bytecode into an array (this is the raw data the program follows).

<img src="images/photo_7.jpg" alt="image" width="100%">

This function is exciting; we see that after decoding, the bytecode is shifted to the left by three and stored in the context. Right after this is finished, we see another call to a function called `Pp`, which takes in the context (`this`) and a static number, `8001`. Let's look into this a bit more. This function is a larger one, again, and we can see a lot of stuff going on here.

<img src="images/photo_8.jpg" alt="image" width="100%">

One thing that stands out to me is that it uses the `8001` and checks to ensure it is greater than 0; this indicates that there is a maximum number of iterations the program can run. Another thing I notice is the `(m >= K)` check. This interests me because it breaks if this condition is met; it suggests that this may be the handler for the control flow, also known as the "CFH". It also seems to retrieve both functions and variables from a call to `J`, with the first argument being the VM context and the next being a number. This should hint at being used for storing some data in the VM (perhaps the registers?), and we see it used to get the `m` variable, which is used in the check before breaking out of the loop. We also use it to obtain the `T` variable, which is invoked if certain conditions are met. Could this be the opcode? Let's figure that out by taking a look at what `T` leads to, so I will add a breakpoint. I figured out that `T` is a function, as you can see below, but it also calls other functions, so I inspected those as well, only to find that they are reading bytes from the bytecode. This tells us that we are now dealing with an opcode for sure.

<img src="images/photo_9.jpg" alt="image" width="100%">

Now that we know this is the control flow handler and it executes the opcodes, we can begin writing the first part of our disassembler. Given that we know the `J` function retrieves the registers, we also understand that the opcodes are stored in registers, which means we may need to track register values, adding more complexity to disassembling this VM. Let's start by identifying which opcodes are used and which are not. You might be thinking we cannot do this due to the anti-logger and anti-debugger, but there's a way around it. We can log a string upon each opcode execution without triggering any anti-logging mechanisms by carefully embedding the logging within each opcode, just a simple line of code at the start; console.log(”opcode #1 is used!”). This method ensures we collect accurate data to eliminate potential 'fake' opcodes, while avoiding any breakages. After doing this, we found multiple opcodes that are not used. Let's create a table below of the ones we know will be used and add labels to them as well.

| Opcode | Name      |
|--------|-----------|
| 328    | USHR      |
| 65     | SETPROP   |
| 191    | NOP       |
| 220    | IN        |
| 307    | LOADOP    |
| 45     | INITPROP  |
| 453    | OR        |
| 263    | LOADARRAY |
| 362    | LOADSTRING|
| 381    | ADD       |
| 326    | NEWARRAY  |
| 283    | TOSTRING  |
| 467    | GETPROP   |
| 289    | HALT      |
| 89     | LOADI32   |
| 348    | TYPEOF    |

We will address this in a moment, but below are the new opcodes created at runtime.

| Opcode | Name      |
|--------|-----------|
| 127    | LOADI8    |
| 91     | LOADI16   |
| 5      | MOV       |

Now, let's begin reversing the functions that read memory or any bytes from the bytecode. I've found two methods of getting bytes, as shown in the screenshot below, which are used in an opcode.

<img src="images/photo_10.jpg" alt="image" width="100%">

With this, we can reverse them; they are simple, which is beneficial for us. However, each also utilizes another function called `H`, which is invoked as follows: `H(true, L, 8)`. This function, as shown below, is the primary function for reading from memory. It reads a given number of bits and, based on the first argument, if it is true, calls some cryptic functions to "encrypt" the bytes before returning them. This also increases another **register variable** as it calls a function `M`, which seems to be a function to set the value of a register. It appears that `87` may be the main byte position register since I also saw another opcode that jumps to another byte address.

<img src="images/photo_11.jpg" alt="image" width="100%">

As I mentioned earlier, the read function increments register 86 at the end, and it operates by incrementing the current byte position by a dynamically changing number. Now that we understand the byte position tracker, we can see that it is stored in registers once again. Here we can see how it encrypts the byte before returning it, note that `K` is the return value, so we see how if `L` is truthy, it gets an array of 3 from register 21 (`J(Z, 21)`) and that seems to be the key that is used to encrypt the next byte and return it. We can also see the usage of `Z.W` and `Z.U`, which we already know `Z.U` is the seed used; so, what is `Z.W`? Well, that is the "position" as it increases linearly, unlike the rest of the values.

<img src="images/photo_12.jpg" alt="image" width="100%">

Upon closer inspection, I can even find an opcode that **changes** all 3 of those encryption "keys", including register 21, Z.W, and Z.U, as you can see. This function is a "SETPROP" opcode, but it also changes the register 21 key array and the position/seed keys. It seems to reset the position key back to `undefined`, and the seed gets re-read from the read function, which changes the seed. The key array (reg 21), however, is altered simply by the opcode performing its normal operations, since it is stored in a register as an array.

<img src="images/photo_13.jpg" alt="image" width="100%">

# Making The Disassembler

We have gone through a significant portion of this project so far, and we now understand how BotGuard functions, allowing us to begin work on a disassembler.

Not quite, though. One more problem we have to work out, **self-modifying opcodes**. We all knew this was coming; they HAD to have something else to hit us with, but we can take it. We see this opcode here, which appears to be a simple "LOADSTRING" opcode. It is, but it loads the stringified version of the new opcodes that it brings in at runtime. As shown in the screenshot below, it generates a string by utilizing two elements. First, it uses register 274, which, in our disassembly so far, is an array of integers. The opcode also uses another array of strings, which is generated at the beginning of the script.

<img src="images/photo_14.jpg" alt="image" width="100%">

Now, we cannot load and use new opcodes solely by the string, so they have an opcode called `EVAL`. However, since it is not used for anything but the evaluation of the opcodes, I called it `LOADOP`. So, I'd like to recap quickly. We know they use registers for both variables and opcodes. To modify their opcodes and introduce new ones, they must set a new register to the evaluated ones. Not too hard to follow. Now, let's implement it in our disassembler. However, let's use a regex rather than a static approach, which will make it better.

We have properly added support for everything needed, but there's still one more thing, outside of classic opcodes. It is an operation, but not quite; it is an error handler, but it's responsible for one of the most essential parts in the VM. It handles the main loop within the VM, utilizing `loop unrolling`, an optimization technique that can be seen as both beneficial and obfuscating, depending on the context. Anyways, they catch errors, then send it off into another function that has a few calls to make, it modifies two different register values (arrays) and checks if another register value (integer) is more than 3, if it is, it subtracts three and calls another encryption function on both the arrays. This encryption function pushes the newly encrypted values into each of the arrays. It does this repeatedly, and the loop count varies per script; however, for this particular script, it takes approximately 38 iterations. Below you can see the error handler function.

<img src="images/photo_15.jpg" alt="image" width="100%">

# Summary

After days of hard work, this project has been completed. You can find the disassembler and decompiler here, as I've left them open-source for you to work with. I've also left the token generation script here for you, which uses the decompiled VM logic that I reversed to generate valid tokens. Keep in mind that this uses static variables, so it only works on the given bytecode sample. To update this script for global use, adjustments to the static variables (parsing) and possibly changes to the logic would be required, depending on the specific changes made. I will not be releasing a JavaScript parser or any tool that allows you to instantly paste this script and cause harm to Google or related sites.

This was a very challenging and enjoyable task; I really look forward to doing more things like this. Let's hope Google updates soon so I can have more fun, and you can try it out for yourself.

I know we see this all the time, but please note that you cannot use this for malicious purposes; it can cause significant damage, so I'm releasing this only to learn from it. I do not want anyone to adapt this for malicious purposes.

# Contact

If you'd like to talk to me (Cypa) about this project or any other matter, please use one of the methods below.

- <a href="mailto:botguard@csolver.net">botguard@csolver.net</a>
- <a href="https://t.me/botguarded">@botguarded</a>
- <a href= "https://t.me/ReversedIt">@ReversedIt</a> This is my (Cypa) community I made on Telegram if you'd like to join

**I (Cypa) do not use Discord for outreach unless explicitly requested.** If someone contacts you claiming to be me (Cypa) on Discord, verify via email or Telegram first.

