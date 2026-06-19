Refine my plan on identifying the UX bugs from posthog sessions. We have post hog running on payment and dashboard pages


```
Payment ppages are scored higher than client dashboard

Before checking any bugs, first we would need how the expected behaviour looks like
- expected the payment page flow
- Expectd events
- Expected time of reponse 
- elemtns that does not cuase a DOM

then we would ge the signals from the posthog
- click events
- dom changes
- network requests
- page transitions

Detect the UX issues
- Dead clicks 
- Rage Cliks 
- Abandmont of forms or cart
- network 500 crash error

Score them
- if payment page +2
- if occured more than 1 time,  +3
- if its rage click +2
- repition +1

question
- what are normal network logs
- what expected flows

```