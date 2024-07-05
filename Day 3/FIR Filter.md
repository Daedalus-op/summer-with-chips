# FIR-Filter
---
[hackster.io](https://www.hackster.io/michi_michi/fpga-fir-filter-hls-kria-kv260-pynq-2eec35)
### Introduction

With the [Vitis High-Level Synthesis (HLS)](https://docs.xilinx.com/r/en-US/ug1399-vitis-hls/Getting-Started-with-Vitis-HLS) the general development time for FPGAs can be shortened considerably.

In this project it will be shown how to accelerate a FIR filter on a FPGA using HLS.

In a previous [blog post](https://www.hackster.io/michi_michi/fpga-deep-learning-inference-hls-kria-kv260-pynq-b0c510) about running a simple Neural Network with HLS, the setup procedure for the KV260 with Pynq has been shown.

All data and pre-built hardware are in the attached [GitHub repository](https://github.com/Nunigan/FIR-FIlter_HLS)

### Fundamentals

In digital signal processing a Finite Impulse Response (FIR) Filter has a finite response to any given finite input signal. A FIR filter is constructed with a tapped delay line for delaying the input signal by a given number of taps (_N_). The $z^{-1}$ is the delay operator from the [Z-Transformation](https://en.wikipedia.org/wiki/Z-transform)<br>
![FIR ref1](../assets/fir_ref1.avif)
<br>The filter coefficients can be arranged in a impulse response vector.<br>
![FIR ref2](../assets/fir_ref2.avif)
<br>The output signal can be computed with<br>
![FIR ref3](../assets/fir_ref3.avif)
<br>or short<br>
![FIR ref4](../assets/fir_ref4.avif)
<br>Which is the same as the convolution of the input signal with the impulse response<br>
![FIR ref5](../assets/fir_ref5.avif)
For the filter design the [Scipy Cookbook about the lowpass FIR-filter design with python](https://scipy-cookbook.readthedocs.io/items/FIRFilter.html) was used.

The filter has been designed with a kaiser window with following properties:

- Cutoff-frequency (_f_c_) of 10 Hz
- Transition width (_∆f_) of 5 Hz
- Stopband ripple (A_stop) of 60 dB
![FIR Reference](../assets/fir_diag1.avif)

> Filter design with Kaiser window. Own presentment, inspired by title={Introduction to signal processing}, publisher={Prentice Hall}, author={Orfanidis, Sophocles J.}, year={1998}

The cookbook has been adapted for this project in [fir.py](https://github.com/Nunigan/FIR-FIlter_HLS/blob/main/src/fir.py)

<br>Coefficients: <br>
![FIR req1](../assets/fir_req1.avif)

<br>Frequency Response: <br>
![FIR req 2](../assets/fir_req2.avif)

<br>Filtered Signal:<br>
![FIR req3](../assets/fir_req3.avif)

<p>The final plots shows the original signal (thin blue line), the filtered signal (shifted by the appropriate phase delay to align with the original signal; thin red line), and the "good" part of the filtered signal (heavy green line). The "good part" is the part of the signal that is not affected by the initial conditions.</p>

In the cookbook the scipy function `scipy.signal.lfilter()` is used for filtering a singal. A pure and non optimized python (with NumPy) implementation would look like:
```python
def fir (x, h):
	nsamples = len(x)
	N = len(h)
	y = np.zeros(len(x))
	shift_reg = np.zeros(len(N))
	
	for j in range(nsamples):
		acc = 0
		for i in range(N-1, 0, -1):
			shift_reg[i] = shift_reg[i - 1]
			acc += shift_reg[i] * h[i]
		acc += x[j] * h[0]
	
		shift_reg[0] = x[j]
		y[j] = acc
	return y
```

### High Level Synthesis

For the HLS part we filter a signal with length 1024 with a 74 tap filter. With no parallelism we need about 75k cycles to filter the signal.

The C++ code in [fir.cpp](https://github.com/Nunigan/FIR-FIlter_HLS/blob/main/src/fir.cpp) for the HLS looks very similar to the Python code. With some code hoisting techniques (put the code when _i = 0_ outside of the for-loop) HLS can pipeline the outermost loop. If a pipelined loop contains more loops, they will be automatically unrolled.

The Python script [fir.py](https://github.com/Nunigan/FIR-FIlter_HLS/blob/main/src/fir.py) writes the computed tap coefficients into a C++ header file. For debugging purpose, a test signal and the expected response are also written into the [fir.h](https://github.com/Nunigan/FIR-FIlter_HLS/blob/main/src/fir.h) header file.

```cpp
void fir(const float input[], float output[]){
#pragma HLS INTERFACE m_axi port=input bundle=gmem0 offset=slave depth=1024
#pragma HLS INTERFACE m_axi port=output bundte=gmem1 offset=slave depth=1024
#pragma HLS INTERFACE s_axilite port=input bundle=control
#pragma HLS INTERFACE s_axilite port=output bundle=control
#pragma HLS INTERFACE s_axitite port=return bundle=control
	static float shift_reg[NUM_TAPS];
	#pragma HLS ARRAY_PARTITION variable=shift_reg complete dim=0
	for (int j =0; j < SIZE; j ++ ) {
		#pragma HLS pipeline II=1
		
		float acc = 0;
		for (int i = NUM TAPS - 1; i >= 0; i--) {
			shift_reg[i] = shift_reg[i - 1];
			acc += shift_reg[i] * taps[i] ;
		}
		acc += input[j] * taps [0];
		shift_reg[0] = input[j];
		output[j] = acc;
   }
}
```

<p>In the post-synthesis report, we see a rather large overhead because even though the loop is pipelined, it takes 1345 cycles to filter a signal of length 1024. This is due to the expensive floating point operations.</p>
![FIR res1](../assets/fir_res1.avif)
<p>To avoid floating point operation the [fixed point package from Vitis HLS](https://docs.xilinx.com/r/en-US/ug1399-vitis-hls/C-Arbitrary-Precision-Fixed-Point-Types-Reference-Information) can be used. In order not to work with fixed point in Python (for communication with Pynq), the input and output of the function is still in float. Input and output must be typecast accordingly. In this project a word width of 32 bits and an integer width of 1 bit is used.</p>

```cpp
void fir_fixed(const float input[], float output[]){
#pragma HLS INTERFACE m_axi port=input bundle=gmem0 offset=slave depth=1024
#pragma HLS INTERFACE m_axi port=output bundte=gmem1 offset=slave depth=1024
#pragma HLS INTERFACE s_axilite port=input bundle=control
#pragma HLS INTERFACE s_axilite port=output bundle=control
#pragma HLS INTERFACE s_axitite port=return bundle=control

	static ap_fixed<W,I> shift_reg_fixed[NUM_TAPS];
	#pragma HLS ARRAY_PARTITION variable=shift_reg_fixed complete dim=0
	
	for (int j =0; j < SIZE; j ++ ) {
		#pragma HLS pipeline II=1
		
		ap_fixed<W,I> acc = 0;
		for (int i = NUM TAPS - 1; i >= 0; i--) {
			shift_reg[i] = shift_reg_fixed[i - 1];
			acc += shift_reg_fixed[i] * taps_fixed[i] ;
		}
		acc += (ap_fixed<W,I>)input[j] * taps_fixed[0];
		shift_reg[0] = (ap_fixed<W,I>)input[j];
		output[j] = (float)acc;
   }
}
```

As one in the report can see, the overhead is nearly gone and with 1058 cycles really close the the optimal Latency of 1024 cycles.<br>

![FIR res2](../assets/fir_res2.avif)

### Vitis HLS & Vivado

As in the previous blog post, generate the hardware with Vitis HLS and Vivado. For the clock frequency use 100 MHz, it can be overclocked afterwards in Pynq.

### Pynq

The pynq code ([fir.ipynb](https://github.com/Nunigan/FIR-FIlter_HLS/blob/main/pynq/fir.ipynb)) is very similar as in the previous blog post. And the system can be overclocked up to 250 MHz

For the plain Python Implementation a huge performance gain of 3160 times has been achieved. For the comparison with _lfilter()_ from scipy (lib) a performance gain of 6.7 times can be achieved.

[Source Code](https://github.com/Nunigan/FIR-FIlter_HLS)