<script lang="ts">
  import { flip } from "svelte/animate";
  import { onMount } from "svelte";

  let contest,
    scoreboard,
    problems,
    problemAssoc,
    frozenSubmissions,
    frozenCopy;
  let loaded: boolean = false;
  let startTime: Date;
  let usernameQueue: [] = [];
  let alreadyCleard: [] = [];
  let selectedUsername: string = "";

  let lastIndex: number;
  let lastUser: string;
  let intId;

  let awardScreen = false;

  const loadData = async () => {
    loaded = false;
    const response = await fetch(
      "https://api.jsonbin.io/v3/b/62161244ca70c44b6ea77726"
    );

    let data = await response.json();
    console.log(data);

    ({ contest, scoreboard, problems, frozenSubmissions } =
      data.record[0].data);

    sortScore();
    loaded = true;

    startTime = new Date(contest.start_time);

    lastIndex = scoreboard.at(-1).position - 1;
    lastUser = scoreboard.at(-1).username;
    frozenCopy = { ...frozenSubmissions };

    problemAssoc = [];

    problems.forEach((element) => (problemAssoc[element.id] = element));

    console.log(problemAssoc);

    console.log(lastIndex);
  };

  function minute(seconds) {
    var minute = Math.floor(seconds / 60);
    seconds -= minute * 60;

    //var second = seconds;

    return minute;
  }

  const sortScore = () => {
    scoreboard.sort((a, b) => {
      if (a.total_score == b.total_score) {
        return a.total_penalty > b.total_penalty;
      }

      return a.total_score < b.total_score;
    });
  };

  const focusTo = (index) => {
    var scrollDiv = document.getElementById(
      "row" +
        (index - 5 > 0
          ? scoreboard.at(index - 5).username
          : scoreboard.at(0).username)
    ).offsetTop;
    window.scrollTo({ top: scrollDiv, behavior: "smooth" });
  };

  function updateScore(
    submission: [],
    letter: string,
    lastSubmission: boolean
  ): boolean {
    /** Parsing JSON string/null values to int  */
    if (isNaN(scoreboard[lastIndex]["problem_" + letter + "_score"])) {
      scoreboard[lastIndex]["problem_" + letter + "_score"] = parseInt(
        scoreboard[lastIndex]["problem_" + letter + "_score"]
      );

      if (isNaN(scoreboard[lastIndex]["problem_" + letter + "_score"])) {
        scoreboard[lastIndex]["problem_" + letter + "_score"] = 0;
      }
    }

    if (isNaN(scoreboard[lastIndex]["problem_" + letter + "_penalty"])) {
      scoreboard[lastIndex]["problem_" + letter + "_penalty"] = parseInt(
        scoreboard[lastIndex]["problem_" + letter + "_penalty"]
      );

      if (isNaN(scoreboard[lastIndex]["problem_" + letter + "_penalty"])) {
        scoreboard[lastIndex]["problem_" + letter + "_penalty"] = 0;
      }
    }

    if (isNaN(scoreboard[lastIndex]["problem_" + letter + "_submission"])) {
      scoreboard[lastIndex]["problem_" + letter + "_submission"] = parseInt(
        scoreboard[lastIndex]["problem_" + letter + "_submission"]
      );

      if (isNaN(scoreboard[lastIndex]["problem_" + letter + "_submission"])) {
        scoreboard[lastIndex]["problem_" + letter + "_submission"] = 0;
      }
    }

    if (isNaN(scoreboard[lastIndex]["total_score"])) {
      scoreboard[lastIndex]["total_score"] = parseInt(
        scoreboard[lastIndex]["total_score"]
      );

      if (isNaN(scoreboard[lastIndex]["total_score"])) {
        scoreboard[lastIndex]["total_score"] = 0;
      }
    }

    if (isNaN(scoreboard[lastIndex]["total_penalty"])) {
      scoreboard[lastIndex]["total_penalty"] = parseInt(
        scoreboard[lastIndex]["total_penalty"]
      );

      if (isNaN(scoreboard[lastIndex]["total_penalty"])) {
        scoreboard[lastIndex]["total_penalty"] = 0;
      }
    }

    /** Parsing Ended */

    if (submission["verdict"] == "1") {
      /** if the submission is correct, then scores and penalties are updated */
      scoreboard[lastIndex]["problem_" + letter + "_score"] =
        scoreboard[lastIndex]["problem_" + letter + "_score"] +
        parseInt(problemAssoc[submission["problem"]]["score"]);
      scoreboard[lastIndex]["total_score"] =
        Number(scoreboard[lastIndex]["total_score"]) +
        Number(scoreboard[lastIndex]["problem_" + letter + "_score"]);

      var subTime: Date = new Date(submission["time"]); // submission time
      var delay: number = (subTime.getTime() - startTime.getTime()) / 1000; // after how many seconds from the beginning the submission was made

      var agerSub = Number(
        scoreboard[lastIndex]["problem_" + letter + "_submission"]
      );
      var newSub =
        Object.keys(frozenCopy[lastUser][letter]).length -
        Object.keys(frozenSubmissions[lastUser][letter]).length +
        1;

      /** Penalty for a problem = submission count till first correct submission * 20 minutes + first correct submission's delay */
      scoreboard[lastIndex]["problem_" + letter + "_penalty"] =
        Number(scoreboard[lastIndex]["problem_" + letter + "_penalty"]) +
        (agerSub + newSub) * 1200;
      scoreboard[lastIndex]["problem_" + letter + "_penalty"] = minute(
        Number(scoreboard[lastIndex]["problem_" + letter + "_penalty"]) + delay
      );
    } else {
      /** if the submission is not correct, penalty remains zero */
      scoreboard[lastIndex]["problem_" + letter + "_score"] = 0;
    }

    if (scoreboard[lastIndex]["problem_" + letter + "_score"] == 0)
      scoreboard[lastIndex]["problem_" + letter + "_penalty"] = 0;

    if (isNaN(scoreboard[lastIndex]["problem_" + letter + "_penalty"]))
      scoreboard[lastIndex]["problem_" + letter + "_penalty"] = 0;

    scoreboard[lastIndex]["total_penalty"] =
      Number(scoreboard[lastIndex]["total_penalty"]) +
      Number(scoreboard[lastIndex]["problem_" + letter + "_penalty"]);

    if (isNaN(scoreboard[lastIndex]["total_score"]))
      scoreboard[lastIndex]["total_score"] = 0;

    if (submission["verdict"] == "1") {
      /** sorting the scoreboard only if the last processed submission is correct */
      sortScore();
    }

    return false;
  }

  const process = () => {
    if (lastIndex < 0) {
      clearInterval(intId);
    }

    focusTo(lastIndex); // scrolling to the correct processing row tile

    Object.keys(frozenSubmissions).length;

    var lastPersonHasSubmission: boolean = false;

    console.log(frozenSubmissions[lastUser]);

    for (var letter of Object.keys(frozenSubmissions[lastUser])) {
      var element = frozenSubmissions[lastUser][letter];

      if (element.length > 0) {
        lastPersonHasSubmission = true;
        break;
      }
    }

    if (lastPersonHasSubmission) {
      console.log("Last person has pending submission");

      var checkNextProblems: boolean = true;

      for (var i = 0; i < Object.keys(problemAssoc).length; i++) {
        var letter: string = problemAssoc[Object.keys(problemAssoc)[i]].letter;

        console.log("Checking letter " + letter);

        console.log(
          letter +
            " has " +
            Object.keys(frozenSubmissions[lastUser][letter]).length +
            " pending submission"
        );

        // check if user er X has AC in any problem

        var checkIfAC: boolean = false;
        var ACIndex: number = 0;

        for (
          var j = 0;
          j < Object.keys(frozenSubmissions[lastUser][letter]).length;
          j++
        ) {
          submission =
            frozenSubmissions[lastUser][letter][
              Object.keys(frozenSubmissions[lastUser][letter])[j]
            ];

          if (submission["verdict"] == "1") {
            // frozenSubmissions[lastUser][letter] = [];
            ACIndex = j;
            checkIfAC = true;
            break;
          }
        }

        // eitar por puran code

        for (
          var j = 0;
          j < Object.keys(frozenSubmissions[lastUser][letter]).length;
          j++
        ) {
          console.log("Processing User: " + lastUser + " Problem: " + letter);
          var submission: [];

          submission =
            frozenSubmissions[lastUser][letter][
              Object.keys(frozenSubmissions[lastUser][letter])[j]
            ];

          console.log(submission);

          checkNextProblems = updateScore(
            submission,
            letter,
            Object.keys(frozenSubmissions[lastUser][letter]).length == 1
          );
          console.log(
            "Next problem e " + (checkNextProblems ? "zao" : "zaiyona")
          );

          var spliceCount: number = 0;

          if (checkIfAC) {
            spliceCount = ACIndex;
          } else {
            spliceCount =
              Object.keys(frozenSubmissions[lastUser][letter]).length - j;
          }

          frozenSubmissions[lastUser][letter].splice(j, spliceCount);

          if (submission["verdict"] == "1") {
            frozenSubmissions[lastUser][letter] = [];
          }
          break;
        }

        if (!checkNextProblems) {
          break;
        }
      }
    } else {
      console.log("No pending submission for " + lastUser);
      lastIndex--;

      focusTo(lastIndex);
    }

    lastUser = lastUser = scoreboard.at(lastIndex).username;
  };

  const resolve = async () => {
    var scrollDiv = document.getElementById("row" + lastUser).offsetTop;
    window.scrollTo({ top: scrollDiv, behavior: "smooth" });

    await new Promise<void>((done) => setTimeout(() => done(), 2000));

    intId = setInterval(process, 1000);
  };

  onMount(() => {
    loadData();
  });
</script>

<link rel="preconnect" href="https://fonts.googleapis.com" />
<link rel="preconnect" href="https://fonts.gstatic.com" />
<link
  href="https://fonts.googleapis.com/css2?family=Prompt&display=swap"
  rel="stylesheet"
/>

<link
  rel="stylesheet"
  type="text/css"
  href="https://gonitzoggo.com/assets/metro/css/metro.css"
/>
<link
  rel="stylesheet"
  type="text/css"
  href="https://gonitzoggo.com/assets/metro/css/metro-4.css"
/>
<link
  rel="stylesheet"
  type="text/css"
  href="https://gonitzoggo.com/assets/metro/css/metro-icons.min.css"
/>
<link
  rel="stylesheet"
  type="text/css"
  href="https://gonitzoggo.com/assets/metro/css/metro-responsive.min.css"
/>
<link
  rel="stylesheet"
  type="text/css"
  href="https://gonitzoggo.com/assets/metro/css/metro-schemes.min.css"
/>
<main class="text-center p-4 mx-0">
  {#if loaded}
    <h1 style="color:white; font-size:2rem">ICPC Resolver Clone</h1>
    <br />
    <h2 style="color:white; font-size:1rem">By Atiab Jobayer</h2>
    <br /><br />
    <div class="grid">
      {#each scoreboard as row, index (row.username)}
        <div
          id="row{row.username}"
          class="row cells12"
          style={`background-color:${
            lastUser === row.username ? "#305c5c" : "#393e46"
          };padding-bottom:0px;height:4.25rem`}
          animate:flip={{ delay: 250 }}
        >
          <div class="cell" style="height:100%;padding-right:10px">
            <h2
              class="text-xl"
              style="font-size:3rem; padding-top:8px; color:white;"
            >
              {index + 1}
            </h2>
          </div>
          <div class="cell colspan11">
            <div>
              <span style="float:left">
                <h3
                  class="align-left"
                  style={`color:${
                    lastUser === row.username ? "whitesmoke" : "white"
                  };font-size:1rem; padding-top:5px;padding-bottom:0px;margin-bottom:12px`}
                >
                  {row.username}
                </h3>
              </span>
              <span style="float:right">
                <div>
                  <p
                    style="color:whitesmoke;font-size:1rem; padding-top:5px;padding-bottom:0px;margin-bottom:12px;padding-right:10px;"
                  >
                    <b>{row["total_score"]}</b>&nbsp;&nbsp;&nbsp;&nbsp;<small
                      >{row["total_penalty"]}</small
                    >
                  </p>
                </div>
              </span>
            </div>

            <div
              class="flex-grid"
              style="padding-bottom:0px; padding-right:10px; padding-top:0px"
            >
              <div class="row cell-auto-size">
                {#each problems as p, index (p.id)}
                  {#if row["problem_" + p.letter + "_score"] > 0}
                    <!-- eita green -->
                    <div
                      class="cell"
                      style="background-color: #519259; border-radius:5px "
                    >
                      {p.letter}<br />
                      <small>
                        {row["problem_" + p.letter + "_penalty"]}
                      </small>
                    </div>
                  {:else if frozenSubmissions[row.username][p.letter] != undefined && frozenSubmissions[row.username][p.letter].length > 0}
                    <!-- eita yellow   -->
                    <div
                      class="cell"
                      style="background-color:#f8cb2e; border-radius:5px"
                    >
                      {p.letter}<br />
                      <small>
                        {row["problem_" + p.letter + "_submission"] > 0
                          ? row["problem_" + p.letter + "_submission"]
                          : 0}&nbsp;&nbsp;&nbsp;+{Object.keys(
                          frozenSubmissions[row.username][p.letter]
                        ).length > 0
                          ? Object.keys(
                              frozenSubmissions[row.username][p.letter]
                            ).length
                          : ""}
                      </small>
                    </div>
                  {:else if row["problem_" + p.letter + "_score"] == "0"}
                    <!-- eita red -->
                    <div
                      class="cell"
                      style="background-color:#ea5455; border-radius:5px;padding-top:10px"
                    >
                      {p.letter}<br />
                    </div>
                  {:else}
                    <div
                      class="cell"
                      style="background-color:grey; border-radius:5px;padding-top:10px"
                    >
                      <!-- eita grey -->
                      {p.letter}
                    </div>
                  {/if}
                {/each}
              </div>
            </div>
          </div>
        </div>
      {/each}
    </div>
    <button
      on:click={resolve}
      class="fixed z-10 px-4 py-2 rounded text-white bg-indigo-500 hover:bg-indigo-600 font-semibold bottom-8 right-8"
    >
      Resolve
    </button>
  {/if}
</main>

<style>
  :root {
    --svelte-rgb: 255, 62, 0;
    font-family: "Prompt", sans-serif;
  }
  main {
    text-align: center;
    padding: 1em;
    max-width: 240px;
    margin: 0 auto;
    background-color: #252525; /**/
  }

  @media (min-width: 640px) {
    main {
      max-width: none;
    }
  }
  *,
  :before,
  :after {
    box-sizing: border-box;
    border-width: 0;
    border-style: solid;
    border-color: #faf5e4;
  }
  :before,
  :after {
    --tw-content: "";
  }
  abbr:where([title]) {
    -webkit-text-decoration: underline dotted;
    text-decoration: underline dotted;
  }
  h1,
  h2,
  h3,
  h4,
  h5,
  h6 {
    font-size: inherit;
    font-weight: inherit;
  }
  a {
    color: inherit;
    text-decoration: inherit;
  }
  b,
  strong {
    font-weight: bolder;
  }
  code,
  kbd,
  samp,
  pre {
    font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas,
      Liberation Mono, Courier New, monospace;
    font-size: 1em;
  }
  small {
    font-size: 80%;
  }
  sub,
  sup {
    font-size: 75%;
    line-height: 0;
    position: relative;
    vertical-align: baseline;
  }
  sub {
    bottom: -0.25em;
  }
  sup {
    top: -0.5em;
  }
  table {
    text-indent: 0;
    border-color: inherit;
    border-collapse: collapse;
  }
  button,
  input,
  optgroup,
  select,
  textarea {
    font-family: inherit;
    font-size: 100%;
    line-height: inherit;
    color: inherit;
    margin: 0;
    padding: 0;
  }
  button,
  select {
    text-transform: none;
  }
  button,
  [type="button"],
  [type="reset"],
  [type="submit"] {
    -webkit-appearance: button;
    background-color: transparent;
    background-image: none;
  }
  :-moz-focusring {
    outline: auto;
  }
  :-moz-ui-invalid {
    box-shadow: none;
  }
  progress {
    vertical-align: baseline;
  }
  ::-webkit-inner-spin-button,
  ::-webkit-outer-spin-button {
    height: auto;
  }
  [type="search"] {
    -webkit-appearance: textfield;
    outline-offset: -2px;
  }
  ::-webkit-search-decoration {
    -webkit-appearance: none;
  }
  ::-webkit-file-upload-button {
    -webkit-appearance: button;
    font: inherit;
  }
  summary {
    display: list-item;
  }
  blockquote,
  dl,
  dd,
  h1,
  h2,
  h3,
  h4,
  h5,
  h6,
  hr,
  figure,
  p,
  pre {
    margin: 0;
  }
  fieldset {
    margin: 0;
    padding: 0;
  }
  legend {
    padding: 0;
  }
  ol,
  ul,
  menu {
    list-style: none;
    margin: 0;
    padding: 0;
  }
  textarea {
    resize: vertical;
  }
  input::-moz-placeholder,
  textarea::-moz-placeholder {
    opacity: 1;
    color: #9ca3af;
  }
  input:-ms-input-placeholder,
  textarea:-ms-input-placeholder {
    opacity: 1;
    color: #9ca3af;
  }
  input::placeholder,
  textarea::placeholder {
    opacity: 1;
    color: #9ca3af;
  }
  button,
  [role="button"] {
    cursor: pointer;
  }
  :disabled {
    cursor: default;
  }
  img,
  svg,
  video,
  canvas,
  audio,
  iframe,
  embed,
  object {
    display: block;
    vertical-align: middle;
  }
  img,
  video {
    max-width: 100%;
    height: auto;
  }
  [hidden] {
    display: none;
  }
  *,
  :before,
  :after {
    --tw-translate-x: 0;
    --tw-translate-y: 0;
    --tw-rotate: 0;
    --tw-skew-x: 0;
    --tw-skew-y: 0;
    --tw-scale-x: 1;
    --tw-scale-y: 1;
    --tw-pan-x: ;
    --tw-pan-y: ;
    --tw-pinch-zoom: ;
    --tw-scroll-snap-strictness: proximity;
    --tw-ordinal: ;
    --tw-slashed-zero: ;
    --tw-numeric-figure: ;
    --tw-numeric-spacing: ;
    --tw-numeric-fraction: ;
    --tw-ring-inset: ;
    --tw-ring-offset-width: 0px;
    --tw-ring-offset-color: #fff;
    --tw-ring-color: rgb(59 130 246 / 0.5);
    --tw-ring-offset-shadow: 0 0 #0000;
    --tw-ring-shadow: 0 0 #0000;
    --tw-shadow: 0 0 #0000;
    --tw-shadow-colored: 0 0 #0000;
    --tw-blur: ;
    --tw-brightness: ;
    --tw-contrast: ;
    --tw-grayscale: ;
    --tw-hue-rotate: ;
    --tw-invert: ;
    --tw-saturate: ;
    --tw-sepia: ;
    --tw-drop-shadow: ;
    --tw-backdrop-blur: ;
    --tw-backdrop-brightness: ;
    --tw-backdrop-contrast: ;
    --tw-backdrop-grayscale: ;
    --tw-backdrop-hue-rotate: ;
    --tw-backdrop-invert: ;
    --tw-backdrop-opacity: ;
    --tw-backdrop-saturate: ;
    --tw-backdrop-sepia: ;
  }
  .container {
    width: 100%;
  }
  @media (min-width: 640px) {
    .container {
      max-width: 640px;
    }
  }
  @media (min-width: 768px) {
    .container {
      max-width: 768px;
    }
  }
  @media (min-width: 1024px) {
    .container {
      max-width: 1024px;
    }
  }
  @media (min-width: 1280px) {
    .container {
      max-width: 1280px;
    }
  }
  @media (min-width: 1536px) {
    .container {
      max-width: 1536px;
    }
  }
  .fixed {
    position: fixed;
  }
  .bottom-8 {
    bottom: 2rem;
  }
  .right-8 {
    right: 2rem;
  }
  .z-10 {
    z-index: 10;
  }
  .mx-auto {
    margin-left: auto;
    margin-right: auto;
  }
  .flex {
    display: flex;
  }
  .table {
    display: table;
  }
  .min-h-screen {
    min-height: 100vh;
  }
  .table-auto {
    table-layout: auto;
  }
  .items-center {
    align-items: center;
  }
  .justify-center {
    justify-content: center;
  }
  .rounded {
    border-radius: 0.25rem;
  }
  .border {
    border-width: 1px;
  }
  .bg-indigo-500 {
    --tw-bg-opacity: 1;
    background-color: rgb(99 102 241 / var(--tw-bg-opacity));
  }
  .p-4 {
    padding: 1rem;
  }
  .px-4 {
    padding-left: 1rem;
    padding-right: 1rem;
  }
  .py-2 {
    padding-top: 0.5rem;
    padding-bottom: 0.5rem;
  }
  .text-2xl {
    font-size: 1.5rem;
    line-height: 2rem;
  }
  .font-semibold {
    font-weight: 600;
  }
  .text-white {
    --tw-text-opacity: 1;
    color: rgb(255 255 255 / var(--tw-text-opacity));
  }
  .hover\:bg-indigo-600:hover {
    --tw-bg-opacity: 1;
    background-color: rgb(79 70 229 / var(--tw-bg-opacity));
  }
  tr {
    border-width: 1px;
    --tw-border-opacity: 1;
    border-color: rgb(229 231 235 / var(--tw-border-opacity));
  }
  td,
  th {
    padding: 0.5rem 1rem;
  }
</style>
