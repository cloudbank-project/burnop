# Monte Carlo profiling

I found this quote on [stack overflow](https://stackoverflow.com/questions/3045556/how-to-profile-my-code/3068045): 
Very amusing and threatens to be useful as well. In response to "How do I profile my code?"


> The standard answer to this question is to use **cProfile**.
> 
> You'll find though that without having your code separated out into methods that cProfile won't give you particularly rich information.
> 
> Instead, you might like to try what another poster here calls Monte Carlo Profiling. To quote from another answer:
> 
> If you're in a hurry and you can manually interrupt your program under the debugger while it's being subjectively slow, 
> there's a simple way to find performance problems.
>
> Just halt it several times, and each time look at the call stack. If there is some code that is wasting some 
> percentage of the time, 20% or 50% or whatever, that is the probability that you will catch it in the act on each 
> sample. So that is roughly the percentage of samples on which you will see it. There is no educated guesswork required. 
> If you do have a guess as to what the problem is, this will prove or disprove it.
>
> You may have multiple performance problems of different sizes. If you clean out any one of them, the remaining ones 
> will take a larger percentage, and be easier to spot, on subsequent passes.
>
> Caveat: programmers tend to be skeptical of this technique unless they've used it themselves. They will say that profilers give 
> you this information, but that is only true if they sample the entire call stack. Call graphs don't give you the same information, 
> because 1) they don't summarize at the instruction level, and 2) they give confusing summaries in the presence of recursion. 
> They will also say it only works on toy programs, when actually it works on any program, and i seems to work better on bigger 
> programs, because they tend to have more problems to find.
>
> It's not orthodox, but I've used it very successfully in a project where profiling using cProfile was not giving me useful output.
>
> The best thing about it is that this is dead easy to do in Python. Simply run your Python script in the 
> interpreter, press [Control-C], note the traceback and repeat a number of times.
>
