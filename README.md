const MESSAGE_MAGIC = [0x41, 0x55, 0x44, 0x31]; // AUD1
const PREAMBLE = [1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0];

const generateBtn = document.getElementById('generateBtn');
const downloadBtn = document.getElementById('downloadBtn');
const audioPlayer = document.getElementById('audioPlayer');
const messageInput = document.getElementById('messageInput');
const freqZeroInput = document.getElementById('freqZero');
const freqOneInput = document.getElementById('freqOne');
const bitDurationInput = document.getElementById('bitDuration');
const sampleRateInput = document.getElementById('sampleRate');

const audioUpload = document.getElementById('audioUpload');
const decodeBtn = document.getElementById('decodeBtn');
const decodedText = document.getElementById('decodedText');
const decodeStatus = document.getElementById('decodeStatus');

function textToBytes(text) {
  return new TextEncoder().encode(text);
}

function bytesToBits(bytes) {
  const bits = [];
  for (const byte of bytes) {
    for (let i = 7; i >= 0; i--) {
      bits.push((byte >> i) & 1);
    }
  }
  return bits;
}

function bitsToBytes(bits) {
  const bytes = [];
  for (let i = 0; i < bits.length; i += 8) {
    const byteBits = bits.slice(i, i + 8);
    if (byteBits.length < 8) break;
    let value = 0;
    for (let j = 0; j < 8; j++) {
      value = (value << 1) | byteBits[j];
    }
    bytes.push(value);
  }
  return new Uint8Array(bytes);
}

function createStringView(view, offset, text) {
  for (let i = 0; i < text.length; i++) {
    view.setUint8(offset + i, text.charCodeAt(i));
  }
}

function createWavBlob(samples, sampleRate) {
  const buffer = new ArrayBuffer(44 + samples.length * 2);
  const view = new DataView(buffer);

  createStringView(view, 0, 'RIFF');
  view.setUint32(4, 36 + samples.length * 2, true);
  createStringView(view, 8, 'WAVE');
  createStringView(view, 12, 'fmt ');
  view.setUint32(16, 16, true);
  view.setUint16(20, 1, true);
  view.setUint16(22, 1, true);
  view.setUint32(24, sampleRate, true);
  view.setUint32(28, sampleRate * 2, true);
  view.setUint16(32, 2, true);
  view.setUint16(34, 16, true);
  createStringView(view, 36, 'data');
  view.setUint32(40, samples.length * 2, true);

  let offset = 44;
  for (let i = 0; i < samples.length; i++) {
    const sample = Math.max(-1, Math.min(1, samples[i]));
    view.setInt16(offset, sample < 0 ? sample * 0x8000 : sample * 0x7fff, true);
    offset += 2;
  }

  return new Blob([buffer], { type: 'audio/wav' });
}

function encodeMessageToBits(messageText) {
  const payload = textToBytes(messageText);
  const messageLength = payload.length;
  const lengthBytes = new Uint8Array(4);
  lengthBytes[0] = (messageLength >> 24) & 0xff;
  lengthBytes[1] = (messageLength >> 16) & 0xff;
  lengthBytes[2] = (messageLength >> 8) & 0xff;
  lengthBytes[3] = messageLength & 0xff;

  const bits = [
    ...PREAMBLE,
    ...bytesToBits(new Uint8Array(MESSAGE_MAGIC)),
    ...bytesToBits(lengthBytes),
    ...bytesToBits(payload),
  ];

  return bits;
}

function buildToneSegment(frequency, sampleRate, durationSeconds, amplitude = 0.7) {
  const samples = [];
  const totalSamples = Math.max(1, Math.floor(sampleRate * durationSeconds));

  for (let i = 0; i < totalSamples; i++) {
    const t = i / sampleRate;
    const envelope = 0.5 - 0.5 * Math.cos((2 * Math.PI * i) / totalSamples);
    const sample = Math.sin(2 * Math.PI * frequency * t) * envelope * amplitude;
    samples.push(sample);
  }

  return samples;
}

function generateAudioFile() {
  const message = messageInput.value;
  const freqZero = Number(freqZeroInput.value) || 800;
  const freqOne = Number(freqOneInput.value) || 1400;
  const bitDuration = Number(bitDurationInput.value) || 0.12;
  const sampleRate = Number(sampleRateInput.value) || 22050;

  const bits = encodeMessageToBits(message);
  const sampleSegments = [];

  for (const bit of bits) {
    const toneFrequency = bit === 1 ? freqOne : freqZero;
    const segment = buildToneSegment(toneFrequency, sampleRate, bitDuration, 0.82);
    sampleSegments.push(...segment);
  }

  const wavBlob = createWavBlob(sampleSegments, sampleRate);
  const url = URL.createObjectURL(wavBlob);
  audioPlayer.src = url;
  audioPlayer.classList.remove('hidden');
  downloadBtn.href = url;
  downloadBtn.classList.remove('hidden');
}

generateBtn.addEventListener('click', generateAudioFile);

audioUpload.addEventListener('change', () => {
  decodeStatus.textContent = audioUpload.files.length ? 'Ready to decode selected file.' : 'No file selected.';
  decodeStatus.classList.remove('error', 'success');
});

decodeBtn.addEventListener('click', () => {
  const file = audioUpload.files[0];
  if (!file) {
    decodeStatus.textContent = 'Please choose a WAV file first.';
    decodeStatus.classList.add('error');
    return;
  }

  const reader = new FileReader();
  reader.onload = () => {
    try {
      const wav = readWavFile(reader.result);
      const decoded = decodeAudioSamples(wav.samples, wav.sampleRate);
      decodedText.value = decoded.text;
      decodeStatus.textContent = decoded.success ? 'Decoded successfully.' : 'Decode failed.';
      decodeStatus.classList.toggle('success', decoded.success);
      decodeStatus.classList.toggle('error', !decoded.success);
    } catch (error) {
      console.error(error);
      decodeStatus.textContent = 'Unable to decode this audio file. Use a generated WAV from this app.';
      decodeStatus.classList.add('error');
    }
  };

  reader.readAsArrayBuffer(file);
});

function readWavFile(arrayBuffer) {
  const view = new DataView(arrayBuffer);
  const riff = String.fromCharCode(...new Uint8Array(arrayBuffer.slice(0, 4)));
  if (riff !== 'RIFF') {
    throw new Error('Not a WAV file');
  }

  const format = String.fromCharCode(...new Uint8Array(arrayBuffer.slice(8, 12)));
  if (format !== 'WAVE') {
    throw new Error('Not a WAVE file');
  }

  let offset = 12;
  let fmtFound = false;
  let dataFound = false;
  let sampleRate = 22050;
  let channelCount = 1;
  let bitDepth = 16;
  let dataOffset = 44;
  let dataLength = 0;

  while (offset + 8 <= view.byteLength) {
    const chunkId = String.fromCharCode(...new Uint8Array(arrayBuffer.slice(offset, offset + 4)));
    const chunkSize = view.getUint32(offset + 4, true);
    const chunkDataOffset = offset + 8;

    if (chunkId === 'fmt ') {
      fmtFound = true;
      sampleRate = view.getUint32(chunkDataOffset + 4, true);
      channelCount = view.getUint16(chunkDataOffset + 2, true);
      bitDepth = view.getUint16(chunkDataOffset + 14, true);
    }

    if (chunkId === 'data') {
      dataFound = true;
      dataOffset = chunkDataOffset;
      dataLength = chunkSize;
    }

    offset += 8 + chunkSize + (chunkSize % 2);
  }

  if (!fmtFound || !dataFound) {
    throw new Error('Missing WAV format or data chunk');
  }

  if (bitDepth !== 16) {
    throw new Error('Only 16-bit PCM WAV files are supported.');
  }

  const dataView = new DataView(arrayBuffer, dataOffset, dataLength);
  const samples = new Float32Array(dataLength / 2);

  for (let i = 0; i < samples.length; i++) {
    const signed = dataView.getInt16(i * 2, true);
    samples[i] = signed / 32768;
  }

  return {
    sampleRate,
    channelCount,
    samples,
  };
}

function measureToneEnergy(segmentSamples, frequency, sampleRate) {
  let energy = 0;
  for (let i = 0; i < segmentSamples.length; i++) {
    const t = i / sampleRate;
    const signal = segmentSamples[i];
    const sampleWave = Math.cos(2 * Math.PI * frequency * t);
    energy += signal * sampleWave;
  }
  return Math.abs(energy);
}

function decodeAudioSamples(samples, sampleRate) {
  const bitDuration = Number(bitDurationInput.value) || 0.12;
  const frequencyZero = Number(freqZeroInput.value) || 800;
  const frequencyOne = Number(freqOneInput.value) || 1400;
  const segmentLength = Math.max(1, Math.floor(sampleRate * bitDuration));

  const allBits = [];
  const totalSegments = Math.floor(samples.length / segmentLength);

  for (let i = 0; i < totalSegments; i++) {
    const start = i * segmentLength;
    const segment = samples.slice(start, start + segmentLength);
    const energyZero = measureToneEnergy(segment, frequencyZero, sampleRate);
    const energyOne = measureToneEnergy(segment, frequencyOne, sampleRate);
    const bit = energyOne > energyZero ? 1 : 0;
    allBits.push(bit);
  }

  const preambleStart = 0;
  const preambleEnd = PREAMBLE.length;
  const localPreamble = allBits.slice(preambleStart, preambleEnd);
  if (!arraysMatch(localPreamble, PREAMBLE)) {
    throw new Error('Preamble not detected.');
  }

  const headerIndex = PREAMBLE.length;
  const magicBits = allBits.slice(headerIndex, headerIndex + 32);
  const magicBytes = bitsToBytes(magicBits);
  const magicText = Array.from(magicBytes).map((value) => String.fromCharCode(value)).join('');

  if (magicText !== 'AUD1') {
    throw new Error('Invalid magic header.');
  }

  const lengthBits = allBits.slice(headerIndex + 32, headerIndex + 32 + 32);
  const lengthBytes = bitsToBytes(lengthBits);
  const messageLength = ((lengthBytes[0] << 24) | (lengthBytes[1] << 16) | (lengthBytes[2] << 8) | lengthBytes[3]) >>> 0;

  const totalPayloadBits = messageLength * 8;
  const payloadStart = headerIndex + 64;
  const payloadBits = allBits.slice(payloadStart, payloadStart + totalPayloadBits);

  if (payloadBits.length < totalPayloadBits) {
    throw new Error('Audio is too short to contain the full message.');
  }

  const messageBytes = bitsToBytes(payloadBits).slice(0, messageLength);
  const decodedText = new TextDecoder().decode(messageBytes);

  return {
    success: true,
    text: decodedText,
  };
}

function arraysMatch(left, right) {
  if (left.length !== right.length) return false;
  for (let i = 0; i < left.length; i++) {
    if (left[i] !== right[i]) return false;
  }
  return true;
}
