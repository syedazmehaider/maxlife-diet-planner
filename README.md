# maxlife-diet-planner
/*
Dietitian AI - Single-file React app (default export component)

What this includes:
- A clean React + Tailwind UI for dietitians to upload prescriptions & lab reports (PDF/image)
- Local OCR option using Tesseract.js (frontend) or placeholder to call backend OCR service
- Form to capture patient metadata (age, sex, weight, height, allergies, preferences)
- Calls to backend endpoints for: /api/ocr, /api/extract, /api/generate-diet
- Example payloads and comments describing server-side responsibilities

How to use:
1. Drop this file into a React + Tailwind project (e.g., create-react-app or Vite + React).
2. Ensure Tailwind is available, or replace classNames with your CSS.
3. Implement backend endpoints described in comments (Node/Express or serverless).
4. Optionally use Tesseract.js in frontend by uncommenting the usage.

If you want, I can also generate the backend Node/Express code (OCR + LLM orchestration),
or a Flutter/Android APK that wraps this UI.
*/

import React, { useState } from "react";

// NOTE: This file assumes Tailwind CSS is available in your project.
// If not, either add Tailwind or replace className values with your own CSS.

export default function DietitianAIApp() {
  const [files, setFiles] = useState([]);
  const [patient, setPatient] = useState({
    name: "",
    age: "",
    sex: "",
    weightKg: "",
    heightCm: "",
    activity: "Sedentary",
    allergies: "",
    preferences: "Vegetarian",
    notes: "",
  });
  const [extractedText, setExtractedText] = useState("");
  const [isProcessing, setIsProcessing] = useState(false);
  const [dietResult, setDietResult] = useState(null);
  const [error, setError] = useState(null);

  function handleFilesChange(e) {
    setFiles(Array.from(e.target.files));
    setExtractedText("");
    setDietResult(null);
    setError(null);
  }

  function updatePatient(field, value) {
    setPatient(prev => ({ ...prev, [field]: value }));
  }

  async function runOCRFrontend(file) {
    // OPTIONAL: frontend OCR using Tesseract.js. Uncomment to use.
    // import('tesseract.js').then(Tesseract => {
    //   return Tesseract.recognize(file, 'eng', { logger: m => console.log(m) })
    //     .then(({ data: { text } }) => text);
    // });
    throw new Error('Frontend OCR not enabled in this build. Use backend OCR or enable Tesseract.js.');
  }

  async function handleExtract() {
    if (!files.length) {
      setError('Please upload at least one prescription/report file.');
      return;
    }
    setIsProcessing(true);
    setError(null);
    setDietResult(null);

    try {
      // Prepare FormData to send files to backend OCR/extraction endpoint
      const fd = new FormData();
      files.forEach((f, idx) => fd.append('file' + idx, f));
      fd.append('patient', JSON.stringify(patient));

      // Replace /api/extract with your server endpoint that runs OCR + structured extraction
      const resp = await fetch('/api/extract', { method: 'POST', body: fd });
      if (!resp.ok) throw new Error(await resp.text());
      const data = await resp.json();

      // Expecting data: { raw_text: string, findings: { labs: [], meds: [], diagnosis: [] } }
      setExtractedText(data.raw_text || JSON.stringify(data.findings, null, 2));
    } catch (err) {
      console.error(err);
      setError('Extraction failed: ' + (err.message || err));
    } finally {
      setIsProcessing(false);
    }
  }

  async function handleGenerateDiet() {
    setIsProcessing(true);
    setError(null);
    setDietResult(null);

    try {
      // Construct payload for diet generation
      const payload = {
        patient,
        extracted_text: extractedText,
        constraints: {
          calorie_target: computeCalorieTarget(patient),
          exclude: (patient.allergies || '').split(',').map(s => s.trim()).filter(Boolean),
          preferences: patient.preferences,
        },
      };

      const resp = await fetch('/api/generate-diet', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload),
      });
      if (!resp.ok) throw new Error(await resp.text());
      const data = await resp.json();

      // Expecting { diet_chart_markdown: string, diet_chart_html: string, structured: {...} }
      setDietResult(data);
    } catch (err) {
      console.error(err);
      setError('Diet generation failed: ' + (err.message || err));
    } finally {
      setIsProcessing(false);
    }
  }

  function computeCalorieTarget(p) {
    // Very rough Mifflin-St Jeor estimate (male/female) — do not rely clinically; backend should re-check.
    const w = Number(p.weightKg) || 70;
    const h = Number(p.heightCm) || 170;
    const age = Number(p.age) || 30;
    const sex = (p.sex || '').toLowerCase();
    let bmr = sex === 'female'
      ? 10 * w + 6.25 * h - 5 * age - 161
      : 10 * w + 6.25 * h - 5 * age + 5;
    const actFactor = p.activity === 'Active' ? 1.55 : p.activity === 'Very Active' ? 1.725 : 1.375; // light default
    const calories = Math.round(bmr * actFactor);
    return calories;
  }

  return (
    <div className="min-h-screen bg-gray-50 p-6">
      <div className="max-w-4xl mx-auto bg-white rounded-2xl shadow p-6">
        <h1 className="text-2xl font-semibold mb-4">Dietitian AI — Prescription & Report → Diet Chart</h1>

        <section className="grid md:grid-cols-2 gap-4">
          <div>
            <label className="block text-sm font-medium text-gray-700">Upload prescription / reports</label>
            <input type="file" multiple accept="image/*,.pdf" onChange={handleFilesChange}
              className="mt-2 block w-full text-sm text-gray-500 file:mr-4 file:py-2 file:px-4 file:rounded file:border-0 file:text-sm file:font-semibold file:bg-indigo-50 file:text-indigo-700" />

            <div className="mt-4 text-sm text-gray-600">
              <div>Files: {files.length}</div>
              {files.slice(0,5).map((f, i) => (
                <div key={i} className="text-xs mt-1">• {f.name} — {(f.size/1024).toFixed(0)} KB</div>
              ))}
            </div>

            <div className="mt-4">
              <button onClick={handleExtract} disabled={isProcessing}
                className="px-4 py-2 rounded bg-indigo-600 text-white hover:bg-indigo-700 disabled:opacity-60">Run OCR & Extract</button>
              <button onClick={() => { setFiles([]); setExtractedText(''); setDietResult(null); }}
                className="ml-3 px-3 py-2 rounded border">Clear</button>
            </div>

            <div className="mt-6">
              <label className="block text-sm font-medium text-gray-700">Extracted text / findings</label>
              <textarea value={extractedText} readOnly rows={10}
                className="mt-2 w-full p-3 border rounded text-sm font-mono bg-gray-50" />
            </div>
          </div>

          <div>
            <label className="block text-sm font-medium text-gray-700">Patient details</label>
            <div className="grid grid-cols-2 gap-2 mt-2">
              <input placeholder="Name" value={patient.name} onChange={e=>updatePatient('name', e.target.value)} className="p-2 border rounded" />
              <input placeholder="Age" value={patient.age} onChange={e=>updatePatient('age', e.target.value)} className="p-2 border rounded" />
              <select value={patient.sex} onChange={e=>updatePatient('sex', e.target.value)} className="p-2 border rounded">
                <option value="">Sex</option>
                <option>Male</option>
                <option>Female</option>
                <option>Other</option>
              </select>
              <select value={patient.activity} onChange={e=>updatePatient('activity', e.target.value)} className="p-2 border rounded">
                <option>Sedentary</option>
                <option>Lightly active</option>
                <option>Active</option>
                <option>Very Active</option>
              </select>
              <input placeholder="Weight (kg)" value={patient.weightKg} onChange={e=>updatePatient('weightKg', e.target.value)} className="p-2 border rounded" />
              <input placeholder="Height (cm)" value={patient.heightCm} onChange={e=>updatePatient('heightCm', e.target.value)} className="p-2 border rounded" />
            </div>

            <div className="mt-3">
              <input placeholder="Allergies (comma separated)" value={patient.allergies} onChange={e=>updatePatient('allergies', e.target.value)} className="w-full p-2 border rounded" />
            </div>
            <div className="mt-3">
              <select value={patient.preferences} onChange={e=>updatePatient('preferences', e.target.value)} className="w-full p-2 border rounded">
                <option>Vegetarian</option>
                <option>Non-Vegetarian</option>
                <option>Vegan</option>
                <option>Keto</option>
                <option>Diabetic-friendly</option>
              </select>
            </div>
            <div className="mt-3">
              <textarea placeholder="Notes / clinical goals (e.g. lose 5 kg, control blood sugar)" value={patient.notes} onChange={e=>updatePatient('notes', e.target
