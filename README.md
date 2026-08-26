# fft

The fast Fourier transform for [sysl](https://github.com/sysl-lang/sysl), generic over the float
width.

Cooley-Tukey, radix 2, in place — beside the O(n²) sum that is the transform's *definition*. The slow
one is shipped rather than kept in the tests, because the fast transform is a rearrangement of it and
a rearrangement is exactly the kind of thing that can be subtly wrong and still produce plausible
numbers.

```
sh/sysl/fft/
    fft.sysl        the transform and its inverse, in place, and why each length is refused
    dft.sysl        the definition, summed term by term
    signal.sysl     making signals, copying them, and the two quantities read off one
    convolve.sysl   circular convolution, through the transform and by its definition
    peaks.sysl      which frequencies a signal actually has energy at, strongest first
    tests.sysl      what all of it claims, run by `sysl test .`
package.hocon       who this package is, and what it needs of the machine
```

The module is **`sh.sysl.fft`**. It depends on nothing outside the standard library —
`sysl.math`, `sysl.math.complex` and `sysl.slices` are the whole of it.

## Using it

Name it in your project's `package.hocon` and `sysl build` fetches it:

```
dependencies {
  fft { git = "github.com/sysl-lang/fft", version = "0.1.0" }
}
```

```
import sh.sysl.fft.{real_signal, fft_of, peaks}

var signal = real_signal([3.0, 1.0, 4.0, 1.0, 5.0, 9.0, 2.0, 6.0])
var spectrum = fft_of(signal).expect("a power-of-two length")

for b in peaks(spectrum) do print(f"bin ${b.k} carries ${b.mag}%.3s")
```

## Generic over the width, which is not decoration

Every body here is written against `F: Float`, so the same transform runs over `Complex[real]` and
over `Complex[f32]`. That is worth having rather than clever: a signal arriving from an audio device
or an ADC is usually `f32` already, and widening it to transform it doubles the storage for accuracy
the input never had.

The angles a butterfly needs come from the loop counters, so a generic body has to make an `F` out of
an integer. `F(k)` is that cast — a conversion to a type *parameter*, resolved once `F` is known.
`sysl.math.Float` needs no `from_int` member and does not have one.

`tests.sysl` runs the fast-against-slow comparison, the round trip and the conjugate symmetry at
**both** widths, which is the claim the bound exists to make: a transform that had quietly specialised
to one width passes every other test in the file.

## What allocates and what does not

`fft` and `ifft` transform the samples where they lie and allocate **nothing**. That is the half a
program on a board would use, and it is why they are the two functions that take a slice rather than
returning one.

Everything that builds a *new* signal needs storage whose size is only known when it is called:
`dft`, `real_signal`, `copy_of`, `fft_of`, `ifft_of`, `magnitudes`, both convolutions and `peaks`. The
package therefore declares `heap = true`, since a capability clause is a property of the package and
not of a function.

There are **no module-level bindings** outside the test file, so nothing here is storage an
initializer has to fill — which is what lets the package go into a `sysl build-c` archive for a
freestanding target, where there is no entry point to run one and no loader to run a constructor.

## What it checks itself against

Nothing here is checked against a value this package produced.

- The transforms of an impulse, a constant and a cosine are known in closed form.
- The four bins of `[1, 2, 3, 4]` can be done on paper.
- The eight anchored bins came out of numpy.
- Every other check is a **property** — linearity, Parseval, the shift theorem, conjugate symmetry,
  the convolution theorem — and each of those properties *is* the definition of the transform, so an
  implementation satisfying all of them and disagreeing with numpy would be a surprise worth having.

The strongest single check is `dft` against `fft`, bin by bin, over a real signal and a complex one.

## Where it came from

This was `guide/fft` in the compiler's own tree — one of the programs written to force a language
decision rather than to demonstrate a finished language. Its findings are all discharged: an operator
that wanted two operand types became `Mul[Rhs = Self]`, a bound that promises behaviour and never a
value became receiver-free trait membership, and the complex arithmetic it once carried is now
`sysl.math.complex`. What was left when the friction ran out was a library, which is this.

One finding turned out to be two, and both are settled now — one was false as recorded, and the
language has since moved under the other.

The guide recorded it as *"`sysl.math.Bits` cannot say how wide `Self` is"*, which is **false**:
`count_ones() + count_zeros()` is the width by construction, at whatever `Self` turned out to be, and
`sysl.math`'s own docstring says that is exactly why it keeps the second of the pair rather than
leaving it to a subtraction.

What actually stopped `reverse_bits` becoming `v.reverse_bits() >> (w - bits)` was the **empty
shift**. A one-sample transform reaches it with `bits` at zero, so the amount is the full width — and
that used to be **undefined behaviour**, on every integer type. It did not trap and it did not answer
zero; it produced garbage that was not stable across compilations, and was observed to differ between
two runs of one binary.

**That is fixed in the language rather than worked around here.** A shift at or past the operand's
width is defined now: zero for `<<` and for an unsigned `>>`, and the sign for a signed `>>`. The
`within`-typed left shift still traps, as `reference/errors.md` promises.

**The loop stays anyway, and the reason is the floor rather than the shift.** This package declares
`sysl = "0.0.79"`, which is older than the rule, so adopting the one-liner would raise what a
consumer needs — for a saving of `bit_width(n)` iterations once per transform. That is a bad trade,
so the loop is the correct code here rather than a workaround. It is still the one place in this
package a target with a different `usize` could go wrong.

## Licence

ISC. See `LICENSE`.
