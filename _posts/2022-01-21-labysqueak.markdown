---
layout: post
title: The Chirp at the Core
mathjax: true
---

<script type="text/javascript">
window.ctx = new (window.AudioContext || window.webkitAudioContext)();
window.togglePlay = function togglePlay(button, playfunc) {
	if (button.osc) {
		button.textContent = "Play";
		button.osc.stop();
		button.osc.disconnect();
		button.osc = null;
	} else {
		button.textContent = "Stop";
		button.osc = ctx.createOscillator();
		button.osc.connect(ctx.destination);
		playfunc(button.osc);
	}
};

window.DEFAULT_LABY = {
	d0: 1,
	h: 1.75,
	b: 0.09,
	kmax: 100
};

window.playSin = function playSin(osc, freq) {
	osc.type = "sine";
	osc.frequency.setValueAtTime(freq, window.ctx.currentTime);
	osc.start();
};

window.playSqueak = function playSqueak(osc, laby) {
	laby = laby || window.DEFAULT_LABY;
	osc.type = "sine";
	const freqs = window.getSqueakFreqs(laby, 100);
	const duration = window.getSqueakDuration(laby);
	const ramp = 0.1;
	osc.frequency.setValueCurveAtTime(freqs, window.ctx.currentTime + ramp, duration);
	osc.start(window.ctx.currentTime + ramp);
	osc.stop(window.ctx.currentTime + ramp + duration);
};

window.getSqueakFreqs = function getSqueakFreqs(laby, N) {
	const d0 = laby.d0;
	const h = laby.h;
	const b = laby.b;
	const kmax = laby.kmax;

	const c = 343;

	D_p = (k) => d0 + b*k;
	F = (t) => Math.hypot(c*t, h) / (2*b*t);
	const t0 = D_p(0) / c;
	const tmax = D_p(kmax) / c;
	freqs = new Float32Array(N);
	for (let i = 0; i < N; ++i) {
		const t = t0 + i/(N-1) * (tmax - t0);
		freqs[i] = F(t);
	}

	return freqs;
};

window.getSqueakDuration = function getSqueakDuration(laby) {
	const b = laby.b;
	const kmax = laby.kmax;
	const c = 343;

	return kmax * 2 * b / c;
};
</script>

Recently I went to a public park that had a brick labyrinth in it. Here's a picture
of the labyrinth taken from above:

![Collingwood Labyrinth](/assets/images/collingwood-labyrinth.jpg)

Some kids there told me that if you clap your hands in the center of the
labyrinth you'll hear a squeak. I didn't believe them, but I didn't disbelieve enough
not to try it. Sure enough, when I clapped I heard a short, faint squeak sound.

What could cause this squeak? The simplest mechanism I can think of is this.
When you clap, a pressure wave emanates out from your hands in all directions at the
speed of sound. Where the wave strikes an obstacle, some amount of the energy will
be absorbed, and some will be reflected back as an echo. In particular, every time
the sound passes over a brick, a tiny amount will be reflected off the edge of the
brick back toward the center of the labyrinth.

TODO: insert visualization of echoes bouncing back.

Each time the sound wave hits the edge of a brick, then, an echo is sent back to
you. Because only a tiny portion of the wave hits the edge of a given brick, the
resulting echo would be very faint. However, because you're in the center of the
labyrinth, the echoes from each brick belonging to the same ring will arrive back
to you at approximately the same time, amplifying the effect.

So the picture is this: when you clap, a pressure wave zips out from the center of
the labyrinth toward the bricks. When it reaches the first ring of bricks, a faint
echo is sent back. A split second later, the wave hits the next ring, and sends
back another echo, and so on. The result is a series of faint echoes heard in rapid
succession.

What would all these echoes sound like? One might think that it would sound like
exactly what it is: a rapid sequence of faint clapping noises. Indeed, this is
what it would sound like if the echoes were spaced far enough apart. If they arrive
with a frequency above 20Hz, however, the individual claps will blend together into
a single tone, the pitch of which will depend on the frequency with which the claps
return.

To calculate the frequency of return of the echoes, let's simplify the problem a bit
by pretending that the clap occurs in the plane of the labyrinth--in other words, at
ground level. Let's also assume that the clap occurs in the exact center of the
labyrinth. Finally, let's assume that the bricks all have the same width $$b$$. The
sound wave in this picture is an ever-widening circle originating in the shared
center of the labyrinth's concentric rings:

TODO: diagram/animation of sound wave in 2D model.

Now, let's label the rings of bricks $$k = 0, 1, 2, ...$$, from the inside out. Call
the distance from the center to the first ring $$d_0$$. The distance to the second
ring is $$d_0 + b$$. In general, the distance to the $$k^{th}$$ row of bricks is
given by:

$$D_\parallel(k) = d_0 + bk$$

I've called this distance $$D_\parallel$$ to emphasize that this is the distance
in the case where the wave is travelling parallel to the ground.

The time $$T_\parallel(k)$$ it takes for the $$k^{th}$$ echo to return is the time
it takes for the sound wave to travel to the $$k^{th}$$ row of bricks and back. In
other words, it is twice the distance to the bricks divided by
$$c \approx 343 m/s$$, the speed of sound:

$$T_\parallel(k) = \frac{2 D_\parallel(k)}{c}$$

The time in between echoes $$k$$ and $$k + 1$$ is given by
$$T_\parallel(k+1) - T_\parallel(k) = \frac{2b}{c}$$. This is the time it takes for
the sound wave to travel across a single brick and back. The inverse of this
quantity is the number of echoes per second, or the frequency $$f_\parallel$$:

$$f_\parallel = \frac{c}{2b}$$

Now, let's estimate that the bricks are about $$9 cm$$ in size. Then
$$f_\parallel \approx 1900 Hz$$, which is well within the range of human hearing. In
fact, this is close to 3 octaves above middle C. Here's what the note sounds like as
a sine wave:

<button onclick="window.togglePlay(this, (osc) => playSin(osc, 1906))">Play</button>

Now, the squeak you actually hear isn't like this. Rather than a single tone, it
sounds more like a fast [glissando](https://wikipedia.org/wiki/glissando) from a
higher pitch to a lower one. The simple 2D model above can't account for this,
because it predicts that the echoes all come back at the same frequency.

We can recover the varying frequency by simply removing the assumption that the clap
occurs at ground level. Let the clap now occur at some height $$h$$ above the
ground (but still in the exact center of the labyrinth). For simplicity, let's
assume that the listener's ear is at the exact same point. Here's an animation of
the clap now:

TODO: animation with side view showing sound wave and echoes

Note that the echoes no longer return at a constant rate: the initial echoes are
bunched up more than subsequent ones.

From the animation, we can see that the distance (no longer in the ground plane) to
the $$k^{th}$$ ring of bricks is related to the ground-level distance
$$D_\parallel$$ by the Pythagorean Theorem:

$$D(k) = \sqrt{D_\parallel(k)^2 + h^2}$$

As before, the time $$T(k)$$ at which the $$k^{th}$$ echo returns is twice the
distance to the bricks divided by the speed of sound:

$$T(k) = \frac{2 D(k)}{c}$$

Now, we could calculate the discrete difference $$T(k+1) - T(k)$$ as we did above,
but the math works out nicer if we treat $$T(k)$$ as a continuous differentiable
function and consider the derivative $$T'(k)$$ instead. This will only give us an
approximate estimate of the true value, but the differences here are small enough
that the approximation is very good.

We can evaluate $$T'(k)$$ using the Chain Rule:

$$T'(k) = \frac{2b}{c} \frac{D_\parallel(k)}{D(k)}$$

The inverse of this quantity gives us the (instantaneous) frequency of the echo
signal after the $$k^{th}$$ echo:

$$F(k) = \frac{c}{2b} \frac{D(k)}{D_\parallel(k)}
= f_\parallel \frac{D(k)}{D_\parallel(k)}$$

From this we see that raising the clap above the labyrinth modulates the
ground-level frequency by the ratio of the true distance to the ground-level
distance. Now, if $$h$$ is small compared to $$D_\parallel$$--in other words, if
the distance to the ring producing an echo is much greater than the clapper's
height--then $$D(k) \approx D_\parallel(k)$$, so $$F(k) \approx f_\parallel$$.
If $$D(k)$$ is about equal to $$h$$, however, $$D(k) > D_\parallel(k)$$, and the
frequency will be higher than the ground-level frequency.

We should therefore expect the initial echoes to be coming in at a relatively high
frequency, and later echoes to even out to the ground-level frequency
$$f_\parallel$$.  The graph below shows $$F(k)$$ for $$d_0 = 1$$, $$h = 1.75$$, and
$$b = 0.09$$.

<iframe src="https://www.desmos.com/calculator/yc6gipgr1h?embed" width="500" height="500" style="border: 1px solid #ccc" frameborder=0></iframe>

Now, if we want to listen to the squeak predicted by this model, we have to express
the frequency in terms of time, rather than our parameter $$k$$. Fortunately,
the only terms in the equation for $$F(k)$$ that depend on $$k$$--and therefore on
time--are $$D(k)$$ and $$D_\parallel(k)$$, and $$D(k)$$ itself is expressed in
terms of $$D_\parallel(k)$$. So, all we need is an expression for
$$D_\parallel(t)$$. We could derive this by first finding an expression for $$k$$ in
terms of $$t$$, but we don't have to do that. Recall that $$D_\parallel$$ is the
distance from the center of the labyrinth to the site of an echo. Since the echo is
produced at the exact time the sound wave arrives at the distance $$D_\parallel$$,
we have the following:

$$D_\parallel(t) = ct$$

In English, the ground-level distance at time $$t$$ is the speed of sound times
$$t$$.

Using this, we can compute $$F(t)$$:

$$F(t) = f_\parallel \frac{D(t)}{D_\parallel(t)}
= \frac{1}{2b} \frac{D(t)}{t}$$

The second equality offers another interpretation of this formula: the frequency is the speed at which the sound wave is travelling ($$\frac{D(t)}{t}$$) divided by the
distance the wave travels between echoes ($$2b$$).

Here's a graph of $$F(t)$$ using the same parameters as above:

<iframe src="https://www.desmos.com/calculator/oombctnpzc?embed" width="500" height="500" style="border: 1px solid #ccc" frameborder=0></iframe>

The dotted line is at $$t_0 = 2d_0/c \approx 0.006s$$, which when the first echo
returns. The graph ends at $$t \approx 0.06s$$, which is the time it would take
for the echo to return from the 100th ring of bricks. The whole squeak lasts about
$$0.05s$$. It starts at a frequency of about $$2500Hz$$, and ends up at around
$$1910Hz$$, close to the asymptotic frequency. In musical terms, the squeak starts
at about the Eb 3 octaves above middle C (the last Eb on a piano keyboard) and drops
about a perfect 4th to the Bb below.

We can get a sense for what this sounds like by modulating a sine wave to match the
frequency profile above. Here's what that sounds like:

<button onclick="togglePlay(this, (osc) => window.playSqueak(osc))">Play</button>

This roughly matches my memory of the squeak: a short, high-pitched sound descending
slightly in pitch. The [timbre](https://en.wikipedia.org/wiki/Timbre) is off, but that's not surprising--a clap is much
more complex than a sine wave!

Some day I'll go back and try to record the squeak so I can check if its frequency
profile matches this prediction.

Problems with the model
-----------------------
There's a serious problem with this model: the amplitude of the first echo will be
*much* higher than that of the last echo. To see why, think about what happens when
the clap occurs. The energy creates a pressure wave that rushes out in all
directions. We can picture the wave as an expanding sphere. Now, the initial energy
of the clap is spread out over the surface of this sphere, and the area of the
surface increases as the square of its radius. The amplitude of the sound should
therefore decrease as the square of the distance from the sound source. Any single
brick at distance $$d$$ from the center of the labyrinth is therefore only
reflecting a signal with strength proportional to $$1/d^2$$ of the original sound.
Now, it's not quite as bad as that, because the number of bricks reflecting the
sound increases linearly with the distance. Given that, we should expect the
strength of the signal reflected off the $$k^{th}$$ ring of bricks, considered as a
whole, to decrease linearly with $$k$$. Suppose the radius of the labyrinth is 10m,
and the inner radius $$d_0$$ is 1m. Then the strength of the signal reflected by the
last ring of bricks is 10x weaker than that reflected by the first.

10x isn't so bad--after all claps are quite loud--but it gets even worse. We can
model the echo as a separate sound source that has an amplitude diminished from the
original according to the inverse-square law. Now, that new sound source has to
travel back to the center of the labyrinth, and **it is subject to that very same
inverse square law**! The strength of the echo by the time it gets back to us is
therefore inversely proportional to the square of the square of the distance--in
other words, to the fourth power. As above, the number of bricks reflecting the
signal increases linearly with the distance, so the strength of the echo off of a
ring of bricks should go down as the cube of the distance. We should therefore
expect the last echo to be 1,000x quieter than the first!
