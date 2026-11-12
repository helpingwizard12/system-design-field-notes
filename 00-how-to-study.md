# How to Read System Design Material

Rule: **read with questions, then rebuild the design yourself.**

## Before reading (2 min)
Write the problem in one line and sketch your own (even naive) design. While reading, notice *why* yours fails.

## While reading: answer 5 questions
1. What problem are they solving, at what scale?
2. What goes wrong? (crashes, network splits, overload, duplicates)
3. What trade-off did they pick, and why?
4. What's the mechanism?
5. What did they decide NOT to do?

## After reading
1. **Draw it from memory.** Where you get stuck is what you didn't understand.
2. **Explain it out loud in 2 minutes**, like an interview.
3. **Break it.** "What if the leader dies here? What if the network splits? What if 10,000 requests arrive at once?" Trace what happens.

## Connect
Add every pattern to the pattern index in the main README. After 10 to 15 readings, the same ~20 patterns keep showing up. Those patterns *are* system design.

## Pace
One chapter or paper per week. Skim first, then a slow second read. Pair with the matching DDIA chapter.
