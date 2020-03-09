<template>
  <section class="state-detail my-3" v-if="district != null">
    <div class="container">
      <div class="row">
        <div class="container">
          <router-link
            :to="{name:'state-detail', params: {state:state_code} }"
          >&laquo; Back to state</router-link>
        </div>
      </div>

      <div class="row">
        <h1 class="col-sm">{{ district.name }} ({{ district_code }} - {{ state_code }})</h1>
      </div>

      <DistrictSummary v-bind:district="district" v-bind:schoolDays="schoolDays" v-bind:editMode="editMode" />

      <!-- <ScenarioControl /> -->
    </div>

    <div class="row">
      <div class="col-sm-12">
        <div class="alert alert-primary" role="alert">
          <strong>PLEASE NOTE</strong>
          The data shown for this district is from the 2018-2019 CALPADS aggregate data, as well as April 2019 meals average meals data.
          ISP numbers are for <strong>Direct Certification Only</strong>. Your district's numbers may be significantly higher. 
          Some charter schools may be included, and some preschool schools may be missing, based upon the school data we receive from CALPADS.
          To get the best recommended grouping, the school listing and ISP numbers should be modified to match the reality of your school.
          For more information or questions, please <router-link to="/contact">Contact Us</router-link>!
        </div>
      </div>
    </div>

    <div class="row container mx-auto" v-if="district != null && viewMode == 'group'">
        <DistrictGroupView v-bind:district="district" />
    </div>

    <div v-if="district != null && viewMode == 'table'">
        <div class="col-sm-12">
              <button
                class="btn btn-primary"
                type="button"
                data-toggle="button"
                aria-pressed="false"
                autocomplete="off"
                v-on:click="toggleEdit"
              >edit</button>

              <button
                class="btn btn-primary"
                type="button"
                data-toggle="button"
                aria-pressed="true"
                autocomplete="off"
                v-on:click="submit"
              >submit</button>

              <button
                class="btn btn-primary"
                type="button"
                data-toggle="button"
                aria-pressed="true"
                autocomplete="off"
                v-on:click="reload"
              >reload</button>

              <span class="optimize_badge badge badge-secondary" v-if="'optimization_info'in district">Optimized on {{ district.optimization_info.timestamp }} in {{ district.optimization_info.time.toFixed(2) }}s</span>

        </div>

        <DistrictSchoolView v-bind:schools="district.schools" v-bind:best_group_index="best_group_index" v-bind:editMode="editMode" />
    </div>

    <div>
      <sup>1</sup>
      Based on {{ schoolDays }} days in school year
      <br />
      <sup>2</sup>Derived from Direct Certified only
      <br />
      <sup>3</sup>From average meals per day April 2019, based upon CFPA SNP Report
      <br />
    </div>
  </section>
</template>

<script>
import * as _ from "lodash";
import DistrictSummary from "./DistrictSummary.vue"
import ScenarioControl from "./ScenarioControl.vue"
import DistrictSchoolView from "./DistrictSchoolView.vue"
import DistrictGroupView from "./DistrictGroupView.vue"

// TODO "404" if no district?
// TODO break this down into little components...
export default {
  components: {
    DistrictSummary,
    ScenarioControl,
    DistrictSchoolView,
  },
  props: ["state_code", "district_code"],
  data() {
    return {
      editMode: false,
      viewMode: "table", // or "group"
      schoolDays: 180,
    }
  },
  computed: {
    district() {
      return this.$store.getters.selected_district;
    },
    edited(){
      return this.district != null && this.district.edited != undefined;
    },
    grouped_schools() {
      if (this.district == null || this.district == null) {
        return [];
      }
      const s = this.district.strategies[this.district.best_index];
      const grouped = [];
      const schools = this.district.schools;
      var i = 1;
      s.groups.forEach(g => {
        const group = {
          id: i,
          data: g,
          schools: _.filter(schools, s => {
            return g.school_codes.indexOf(s.school_code) >= 0;
          })
        };
        i++;
        grouped.push(group);
      });
      return _.orderBy( grouped, ['data.isp'], 'desc');
    },
    best_strategy() {
      if (this.district == null || this.district == null) {
        return null;
      }
      if (!this.district.strategies) {
        return null;
      }
      return this.district.strategies[this.district.best_index];
    },
    best_group_index() {
      if (this.best_strategy == null) {
        return null;
      }
      const group_index = {};
      var i = 1;
      this.best_strategy.groups.forEach(g => {
        g.school_codes.forEach(sc => {
          group_index[sc] = i;
        });
        i += 1;
      });
      return group_index;
    }
  },
  methods: {
    toggleEdit() {
      this.editMode = !this.editMode;
    },
    submit(){
      console.log("Submitting for optimization",this.district) 
      this.$store.dispatch("run_district",this.district);
    },
    reload(){
      this.$store.dispatch("load_district",{code:this.district.code,state:this.district.state_code});
    },
    daily_reimbursement_by_school ( school_code ){
      if(this.district_form.reimbursement_rates == null){
        return "";
      }
      return "?"

      // TODO - how do we determine group rate for this school?
      // if group ISP < 0.4, $0
      // if group ISP < 0.625, paid rate
      // else free rate
      const s = this.school_form[school_code];
      const v = ( s.daily_breakfast_served * this.district_form.reimbursement_rates.free_bfast ) 
                 + ( s.daily_lunch_served * this.district_form.reimbursement_rates.free_lunch )
      return v;
    },
  },
};
</script>

<style scoped>
tr.inactive {
  font-style: italic;
  color: #999;
}
input {
  width: 4em;
}

tr.add_row {
  background-color: #eee;
}
tr.add_row td input {
  width: 100%;
}
.optimize_badge {
  animation: fadeInAnimation ease 1s;
  animation-iteration-count: 1; 
  animation-fill-mode: forwards; 
} 
  
@keyframes fadeInAnimation { 
    0% { opacity: 0; } 
    100% { opacity: 1; } 
} 
</style>