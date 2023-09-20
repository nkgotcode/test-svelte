<script>
  import init, { process_audio } from "../audio-process/pkg/audio_process";
  document.addEventListener("DOMContentLoaded", () => {
    const audioFileInput = document.getElementById("audioFileInput");
    const audioPlayer = document.getElementById("audioPlayer");

    audioFileInput.addEventListener("change", handleFileUpload);

    function handleFileUpload(event) {
      const file = event.target.files[0];
      if (file) {
        const objectURL = URL.createObjectURL(file);
        audioPlayer.src = objectURL;
        const asset = AV.Asset.fromFile(file);
        asset.get("format", function (format) {
          const sampleRate = format.sampleRate;
          processAudioJS(objectURL, sampleRate);
        });
      }
    }

    function processAudioJS(audioURL, sampleRate) {
      const audioContext = new (window.AudioContext ||
        window.webkitAudioContext)();
      const audioSource = audioContext.createBufferSource();

      fetch(audioURL)
        .then((response) => response.arrayBuffer())
        .then((buffer) => {
          // Use the Web Audio API to decode the WAV audio data
          audioContext.decodeAudioData(buffer, (audioBuffer) => {
            // 'audioBuffer' is now a decoded AudioBuffer object
            const audioData = audioBuffer.getChannelData(0); // Get the first channel's data as a Float32Array
            const numChannels = audioBuffer.numberOfChannels;
            // Now you can use 'audioData' as a Float32Array for processing
            const wasmMemoryStart = new Float32Array(audioBuffer);
            // const memoryPages = Math.ceil(audioData.length / 65536); // Calculate memory pages
            // const sampleRate = audioBuffer.sampleRate;
            // console.log("sample rate: ", sampleRates);
            // Ensure that 'wasmMemoryStart' is a valid Float32Array
            if (wasmMemoryStart instanceof Float32Array) {
              // Check the bounds before setting data
              if (audioData.length <= wasmMemoryStart.length) {
                // Copy the audio data (Float32Array) into the WebAssembly memory
                wasmMemoryStart.set(audioData);
                // console.log("wasmMemoryStart:", wasmMemoryStart);
              } else {
                console.error(
                  "Audio data exceeds the bounds of wasmMemoryStart"
                );
              }
            } else {
              console.error("wasmMemoryStart is not a Float32Array");
            }

            // Initialize WebAssembly
            init().then(() => {
              process_audio(wasmMemoryStart, sampleRate, numChannels);
            });
          });
        })
        .catch((error) => console.error("Error decoding audio:", error));
    }
  });
</script>

<main>
  <input type="file" id="audioFileInput" />
  <audio id="audioPlayer" controls />
  <script src="/build/aurora-bundle.js"></script>
</main>

<style>
  main {
    text-align: center;
    padding: 1em;
    max-width: 240px;
    margin: 0 auto;
  }

  @media (min-width: 640px) {
    main {
      max-width: none;
    }
  }
</style>
