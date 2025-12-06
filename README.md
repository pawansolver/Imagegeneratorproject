const themeToggle = document.querySelector(".theme-toggle");
const promptForm = document.querySelector(".prompt-form");
const promptInput = document.querySelector(".prompt-input");
const promptBtn = document.querySelector(".prompt-btn");
const modelSelect = document.querySelector("#model-select");
const countSelect = document.querySelector("#count-select");
const ratioSelect = document.querySelector("#ratio-select");
const gridGallery = document.querySelector(".gallery-grid");

const API_KEY = "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";

const examplePrompts = [
  "A magic forest with glowing plants and fairy homes among giant mushrooms",
  "An old steampunk airship floating through golden clouds at sunset",
  "A future Mars colony with glass domes and gardens against red mountains",
];

// ===============================
// DARK MODE INIT
// ===============================
(() => {
  const savedTheme = localStorage.getItem("theme");
  const prefersDark = window.matchMedia("(prefers-color-scheme: dark)").matches;

  const isDark = savedTheme === "dark" || (!savedTheme && prefersDark);
  document.body.classList.toggle("dark-theme", isDark);
  themeToggle.querySelector("i").className = isDark
    ? "fa-solid fa-sun"
    : "fa-solid fa-moon";
})();

const toggleTheme = () => {
  const isDark = document.body.classList.toggle("dark-theme");
  localStorage.setItem("theme", isDark ? "dark" : "light");
  themeToggle.querySelector("i").className = isDark
    ? "fa-solid fa-sun"
    : "fa-solid fa-moon";
};

// ===============================
// GET IMAGE DIMENSIONS
// ===============================
const getImageDimensions = (aspectRatio, baseSize = 512) => {
  const [w, h] = aspectRatio.split("/").map(Number);
  const factor = baseSize / Math.sqrt(w * h);

  let W = Math.floor((w * factor) / 16) * 16;
  let H = Math.floor((h * factor) / 16) * 16;

  return { width: W, height: H };
};

// ===============================
// UPDATE IMAGE CARD
// ===============================
const updateImageCards = (index, imgurl) => {
  const imgCard = document.getElementById(`img-card-${index}`);
  if (!imgCard) return;

  imgCard.classList.remove("loading");

  imgCard.innerHTML = `
    <img src="${imgurl}" class="result-img">
    <div class="img-overlay">
      <a href="${imgurl}" class="img-download-btn" download="${Date.now()}.png">
        <i class="fa-solid fa-download"></i>
      </a>
    </div>
  `;
};

// ===============================
// GENERATE IMAGE (FIXED)
// ===============================
const generateImage = async (selectedModel, imageCount, aspectRatio, promptText) => {
  const MODEL_URL = `https://router.huggingface.co/hf-inference/models/${selectedModel}`;
  const { width, height } = getImageDimensions(aspectRatio);

  const imagePromises = Array.from({ length: imageCount }, async (_, i) => {
    try {
      const response = await fetch(MODEL_URL, {
        method: "POST",
        headers: {
          Authorization: `Bearer ${API_KEY}`,
          "Content-Type": "application/json",
        },
        body: JSON.stringify({
          inputs: promptText,
          parameters: { width, height },
          options: { wait_for_model: true },
        }),
      });

      if (!response.ok) {
        const err = await response.json();
        throw new Error(err?.error || "API Error");
      }

      // convert to blob
      const resultBlob = await response.blob();
      const imgURL = URL.createObjectURL(resultBlob);

      updateImageCards(i, imgURL);
    } catch (error) {
      console.log(error);
    }
  });

  await Promise.allSettled(imagePromises);
};

// ===============================
// CREATE IMAGE CARDS
// ===============================
const createImageCards = (selectedModel, imageCount, aspectRatio, promptText) => {
  gridGallery.innerHTML = "";

  for (let i = 0; i < imageCount; i++) {
    gridGallery.innerHTML += `
      <div class="img-card loading" id="img-card-${i}" style="aspect-ratio:${aspectRatio};">
        <div class="status-container">
          <div class="spinner"></div>
          <i class="fa-solid fa-triangle-exclamation"></i>
          <p class="status-text">Generating...</p>
        </div>
      </div>
    `;
  }

  generateImage(selectedModel, imageCount, aspectRatio, promptText);
};

// ===============================
// FORM SUBMIT HANDLER
// ===============================
const handleFormSubmit = (e) => {
  e.preventDefault();

  const selectedModel = modelSelect.value;
  const imageCount = Number(countSelect.value);

  let aspectRatio = ratioSelect.value || "1/1";

  const promptText = promptInput.value.trim();
  if (!promptText) return;

  createImageCards(selectedModel, imageCount, aspectRatio, promptText);
};

// ===============================
// RANDOM PROMPT BUTTON
// ===============================
promptBtn.addEventListener("click", () => {
  const randomPrompt = examplePrompts[Math.floor(Math.random() * examplePrompts.length)];
  promptInput.value = randomPrompt;
  promptInput.focus();
});

// EVENT LISTENERS
promptForm.addEventListener("submit", handleFormSubmit);
themeToggle.addEventListener("click", toggleTheme);
