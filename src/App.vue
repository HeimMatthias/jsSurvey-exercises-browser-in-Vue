<template>
  <button @click="previousSurvey">⯇</button>
  <button @click="nextSurvey">⯈</button>
  <label
    >Choose an exercise: <select
      id="selectElement"
      @change="changeSurvey(survey, $event.target.value)"
    >
      <optgroup
        v-for="group in surveyGroups"
        :label="group.label"
        :key="group.label"
      >
        <option
          v-for="survey in group.surveys"
          :key="survey.value"
          :value="survey.value"
          :disabled="!survey.exists"
          :class="survey.class"
          :data-link="survey.link"
          :data-context="survey.context"
        >
          {{ survey.name }}
        </option>
      </optgroup>
    </select></label
  >
  <button @click="toggleEditMode">Toggle Edit Mode</button>
  <button @click="clearSurveyData">Clear Data</button>
  <button @click="saveSurveyDataToFile">Save Data to JSON file</button>
  <label
    >Select a file to load user input from: <input
      type="file"
      @change="loadSurveyDataFromFile"
      :accept="fileAccept"
  /></label>
  <SurveyComponent :model="survey" />
  <footer>
    <img src="../public/hep-logo.svg" alt="hep Verlag AG Logo"/>
    <p>See <a :href="selectedSurveyLink" target="_blank" rel="noreferrer noopener">exercise in production</a> and in <a :href="selectedSurveyContext" target="_blank" rel="noreferrer noopener">context</a>. (requires license)
    <br/>© <a href="https://www.hep-verlag.ch">hep Verlag AG 2024</a></p>
  </footer>
</template>
<script setup lang="ts">
import { ref, onMounted, computed } from "vue";
import { Model } from "survey-core";
import { SurveyComponent } from "survey-vue3-ui";
import 'survey-core/defaultV2.min.css';
import "./style.css";

// save survey data to localStorage, called on every change
function saveSurveyData(survey) {
  const surveyData = JSON.stringify(survey.data);
  if (surveyData != surveyEmptyValue.value) {
    for (const option of selectElement.options) {
      if (option.value == selectedSurvey.value) {
        option.classList.add("modified");
        break;
      }
    }
    window.localStorage.setItem(selectedSurvey.value, surveyData);
  }
}

// load a different survey.json
function changeSurvey(survey, selectedSurveyValue) {
  saveSurveyData(survey);
  survey.clear();
  selectedSurvey.value = selectedSurveyValue;
  fetch(`exercises/${selectedSurveyValue}.json`)
    .then(function (u) {
      return u.json();
    })
    .then(function (json) {
      console.log(json);
      survey.fromJSON(json); // Update the survey model with new JSON data
      survey.clear();
      surveyEmptyValue.value = JSON.stringify(survey.data); // later, value will only be stored if it differs from this initial value
      // Restore survey results from local storage
      const prevData =
        window.localStorage.getItem(selectedSurvey.value) || null;
      if (prevData) {
        const data = JSON.parse(prevData);
        survey.data = data;
      }
      // adjust footer links
      selectedSurveyLink.value=selectElement.options[selectElement.selectedIndex].dataset.link;
      selectedSurveyContext.value=selectElement.options[selectElement.selectedIndex].dataset.context;
    });
}

function previousSurvey() {
  if (selectElement.selectedIndex > 0) {
    selectElement.selectedIndex -= 1;
    const selectedOption = selectElement.options[selectElement.selectedIndex];
    changeSurvey(survey, selectedOption.value);
  }
}

function nextSurvey() {
  if (selectElement.selectedIndex < selectElement.options.length - 1) {
    selectElement.selectedIndex += 1;
    const selectedOption = selectElement.options[selectElement.selectedIndex];
    changeSurvey(survey, selectedOption.value);
  }
}

function toggleEditMode() {
  survey.mode = survey.mode === "edit" ? "display" : "edit";
}

function clearSurveyData() {
  survey.clear();
  window.localStorage.removeItem(selectedSurvey.value);
  selectElement.options[selectElement.selectedIndex].classList.remove(
    "modified"
  );
}

function saveSurveyDataToFile() {
  const data = survey.data;
  const blob = new Blob([JSON.stringify(data, null, 2)], {
    type: "application/json",
  });
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  const fileName = `userdata.${selectedSurvey.value}.json`; // Use the survey value for the file name
  a.href = url;
  a.download = fileName;
  a.click();
  URL.revokeObjectURL(url);
}

function loadSurveyDataFromFile(event) {
  const file = event.target.files[0];
  if (file) {
    const reader = new FileReader();
    reader.onload = function (e) {
      const data = JSON.parse(e.target.result as string);
      survey.data = data;
    };
    reader.readAsText(file);
  }
}

// init an empty survey
const survey = new Model({});
const surveyGroups = ref([]);
const selectedSurvey = ref("");
const selectedSurveyLink = ref("");
const selectedSurveyContext = ref("");
const surveyEmptyValue = ref("{}");

// load the different exercises into the select-dropdown
async function fetchSurveyGroups() {
  // load sample user data into local storage
  const responseUserData = await fetch("exercises/user-data-sample.json");
  const userData = await responseUserData.json();
  for (const key in userData) {
  if (userData.hasOwnProperty(key)) {
      window.localStorage.setItem(key, userData[key]);
    }
  }

  // load exercise meta-data from json and populate select-dropdown
  const response = await fetch("exercises/exercises.json");
  const data = await response.json();
  const groups = new Set();
  data.forEach((el) => groups.add(el.plattform));
  const newSurveyGroups = new Array();
  for (const group of Array.from(groups)) {
    const groupSurveys = new Array();
    for (const el of data) {
      if (el.plattform == group) {
        const baseUrl = {
          mySkillbox:"https://app.myskillbox.ch/",
          myKV:"https://app.mykv.ch/",
          myDHA:"https://app.dha.mydetailhandel.ch/",
          myDHF:"https://app.dhf.mydetailhandel.ch/"
        }[group];
        groupSurveys.push({
          value: group + "/" + el.surveyId,
          name: el.title,
          exists: true,
          class: (localStorage.getItem(group + "/" + el.surveyId) !== null?"modified":""),
          link: (baseUrl+"survey/"+el.surveyId),
          context: (baseUrl+"module/"+el.moduleSlug+"#"+el.surveyId)
        });
      }
    }
    newSurveyGroups.push({
      label: group,
      surveys: groupSurveys,
    });
  }
  surveyGroups.value = newSurveyGroups;

  // Set the default selected survey and call changeSurvey
  if (
    surveyGroups.value.length > 0 &&
    surveyGroups.value[0].surveys.length > 0
  ) {
    const defaultSurvey = surveyGroups.value[0].surveys[0].value;
    changeSurvey(survey, defaultSurvey);
  }
}

// change the file-ending for the accepted userdata.json file, basically to speed up selecting the correct file.
const fileAccept = computed(
  () => `.${selectedSurvey.value.replace(/\//, "_")}.json`
);
onMounted(() => {
  fetchSurveyGroups(); // Fetch survey groups when the component is mounted
});

// Save survey results to the local storage
// commented out because we do not want to store data after every modification
// survey.onValueChanged.add(saveSurveyData);
// survey.onCurrentPageChanged.add(saveSurveyData);

// call the next exercise upon completion of the survey
survey.onComplete.add(() => {
  nextSurvey();
});
</script>
