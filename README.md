# Analog-to-digital-path
MATLAB simulation of Analog-to-Digital Conversion (ADC) pipeline: analog signal generation, sampling, quantization, encoding, and bitstream representation. Includes experiments with different sampling rates and quantization levels for IoT sensor data acquisition.
1. Analog Signal Generation

In MATLAB, generate a simple continuous-time signal (e.g., sine wave).
%% From Analog to Digital: end-to-end simulation
% Relates each step to how an IoT sensor + ADC works.

clear; close all; clc;

%% PARAMETERS (edit these for your homework experiments)
f_sig   = 100;      % signal frequency (Hz)
Fs      = 1000;     % sampling frequency (Hz) -> try 150, 200, 1000, 5000 ...
bits    = 4;        % ADC resolution (bits)   -> try 3, 4, 6
T_end   = 0.01;     % duration (s)
FS_min  = -1;       % ADC full-scale min (e.g., 0..Vref mapped to -1..+1 after conditioning)
FS_max  = +1;       % ADC full-scale max

%% 1) "Analog" signal (conceptual: very fine time grid)
t_fine   = 0:1e-5:T_end;                     % very fine step ~ "continuous-like"
x_analog = sin(2*pi*f_sig*t_fine);           % example: sine wave

figure; plot(t_fine, x_analog, 'LineWidth', 1.5);
grid on; title('Analog Signal (Sine)'); xlabel('Time (s)'); ylabel('Amplitude');

%% 2) Sampling
Ts        = 1/Fs;
n         = 0:Ts:T_end;
x_sampled = sin(2*pi*f_sig*n);

figure; 
plot(t_fine, x_analog, 'LineWidth', 1.0); hold on;
stem(n, x_sampled, 'filled'); grid on;
title(sprintf('Sampling: f_{sig}=%g Hz, F_s=%g Hz', f_sig, Fs));
xlabel('Time (s)'); ylabel('Amplitude');
legend('Analog (reference)','Samples');

% Nyquist note: need Fs >= 2*f_sig to avoid aliasing.
% Try lowering Fs below 2*f_sig to see distortion (aliasing).

%% 3) Quantization (uniform mid-tread on fixed full-scale)
L        = 2^bits;                   % number of quantization levels
q_step   = (FS_max - FS_min)/L;      % quantization step (LSB)
% Clip to ADC input range (what real ADC front-ends do):
x_clipped = min(max(x_sampled, FS_min), FS_max - eps); % eps avoids index hitting L

% Map to indices [0 .. L-1]
x_index  = floor( (x_clipped - FS_min) / q_step );
% Map back to quantized amplitude (level centers)
x_quant  = (x_index + 0.5)*q_step + FS_min;

figure; 
stem(n, x_sampled, 'filled'); hold on;
stem(n, x_quant, 'filled');
grid on; title(sprintf('Quantization (%d-bit)', bits));
xlabel('Time (s)'); ylabel('Amplitude');
legend('Sampled','Quantized');

% Quantization error & SQNR estimate
q_err = x_sampled - x_quant;
SQNR_dB = 20*log10( rms(x_sampled) / (rms(q_err) + eps) );
fprintf('Estimated SQNR ≈ %.2f dB (rule-of-thumb ~ 6.02*bits + 1.76 = %.2f dB)\n', ...
    SQNR_dB, 6.02*bits + 1.76);

figure;
stem(n, q_err, 'filled'); grid on;
title('Quantization Error'); xlabel('Time (s)'); ylabel('Error');

%% 4) Encoding (binary words)
% Here we encode the *indices* (unsigned), which is exactly what many ADCs output.
binary_codes = dec2bin(x_index, bits);  % char array [Nsamples x bits]

disp('--- First 10 encoded samples (indices as binary) ---');
disp(binary_codes(1:min(10,size(binary_codes,1)), :));

%% 5) Bitstream (serialize the words)
bitstream = reshape(binary_codes.', 1, []);  % row char vector of '0'/'1'
disp('--- First 40 bits of the stream ---');
disp(bitstream(1:min(40, numel(bitstream))));

%% Summary
fprintf('\nSimulation complete!\n');
fprintf('Analog -> Sampling -> Quantization -> Binary Encoding -> Digital Stream\n');
fprintf('Signal freq: %.1f Hz | Fs: %.1f Hz | bits: %d | duration: %.4f s\n', f_sig, Fs, bits, T_end);
fprintf('Total samples: %d | Bits per sample: %d | Total bits: %d\n', ...
    numel(x_sampled), bits, numel(bitstream));
