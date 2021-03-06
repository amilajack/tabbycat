<script>
import _ from 'lodash'

import ConflictUtilitiesMixin from './ConflictUtilitiesMixin.vue'

export default {
  // An item that can be set into a conflicted state; either due to hover events
  // or due to in-panel conflicts. It does so by receiving a list of conflicts
  // from the global event hub and then checking if it matches with any of them.
  // It then sets an internal record or the conflicted state
  mixins: [ConflictUtilitiesMixin],
  data: function () {
    return {
      verboseDebugMode: false,
      isConflicted: {
        hover: {
          team: false, adjudicator: false, institution: false, histories: false,
        },
        panel: {
          team: false, adjudicator: false, institution: false, histories: false,
        },
      },
    }
  },
  mounted: function () {
    // Subscribe to conflict issued events on the main event hub
    // Each event is specific to the type of conflict but not to this instance
    // These looklike 'set-conflicts-for-team (teamID, hoverOrPanel, ...)

    // Subscribe to set/unset events for this type (i.e. 'adj')
    this.$eventHub.$on(`set-conflicts-for-${this.conflictableType}`, this.setConflicts)
    this.$eventHub.$on(`unset-conflicts-for-${this.conflictableType}`, this.setConflicts)
    this.$eventHub.$on(
      `reset-conflicts-for-${this.conflictableType}-${this.conflictable.id}`,
      this.resetConflicts
    )

    // Subscribe to set/unset events for institutions sent by hovers
    if (this.conflictable.institution !== null) {
      this.$eventHub.$on('set-conflicts-for-institution', this.setInstitutionConflicts)
      this.$eventHub.$on('unset-conflicts-for-institution', this.setInstitutionConflicts)
    }
  },
  computed: {
    hasHistoryConflict: function () {
      // Used for the inline template element showing how long ago the seen was
      if (this.isConflicted.hover.histories) {
        return this.isConflicted.hover.histories
      } else if (this.isConflicted.panel.histories) {
        return this.isConflicted.panel.histories
      }
      return false
    },
    conflictableType: function () {
      if (!_.isUndefined(this.team)) {
        return 'team'
      }
      if (!_.isUndefined(this.adjudicator)) {
        return 'adjudicator'
      }
      return null
    },
    conflictable: function () {
      if (!_.isUndefined(this.team)) {
        return this.team
      }
      if (!_.isUndefined(this.adjudicator)) {
        return this.adjudicator
      }
      return null
    },
    conflictingInstitutionIDs: function () {
      if (this.conflictableType === 'team') {
        return [this.conflictable.institution.id]
      }
      // Adjs don't necessarily conflict with their own instutitions
      return _.map(this.conflictable.conflicts.clashes.institution, 'id')
    },
    conflictsStatus: function () {
      let conflictsCSS = 'conflictable'
      if (this.isHovering) {
        conflictsCSS += ' vue-is-hovering'
      }
      _.forEach(this.isConflicted, (types, hoverOrPanel) => {
        _.forEach(types, (status, type) => {
          // Iterate over panel and hover states
          if (status !== false) {
            conflictsCSS += ` ${hoverOrPanel}-${type}`
            if (type === 'histories') {
              conflictsCSS += `-${status}-ago`
            }
          }
        })
      })
      return conflictsCSS
    },
  },
  methods: {
    issueInstitutionalConflictForTeam (state) {
      if (!_.isUndefined(this.conflictable.institution)) {
        // Teams dont have institutional clashes; must set manually
        if (this.conflictable.institution !== null) {
          const id = this.conflictable.institution.id
          if (state === true) {
            this.sendConflict(
              { id: id }, 'institution', 'institution',
              'hover', 'clashes', 'team'
            )
          } else {
            this.unsendConflict(
              { id: id }, 'institution', 'institution',
              'hover', 'clashes', 'team'
            )
          }
        }
      }
    },
    showHoverConflicts: function () {
      this.isHovering = true
      if (this.verboseDebugMode) {
        console.debug(
          'Conflictable showHoverConflicts() for',
          this.conflictableType, this.conflictable.id, this.isConflicted
        )
      }
      // Issue conflict events; typically on beginning hover
      const self = this
      this.forEachConflict(
        this.conflictable.conflicts,
        (conflict, type, clashOrHistory) => {
          self.sendConflict(
            conflict, type, type, 'hover', clashOrHistory,
            self.conflictableType
          )
        }
      )
      if (this.conflictableType === 'team') {
        this.issueInstitutionalConflictForTeam(true)
      }
    },
    hideHoverConflicts: function () {
      this.isHovering = false
      // Issue hide conflict events; typically on ending hover
      const self = this
      this.forEachConflict(
        this.conflictable.conflicts,
        (conflict, type, clashOrHistory) => {
          self.unsendConflict(
            conflict, type, type, 'hover', clashOrHistory,
            self.conflictableType
          )
        }
      )
      if (this.conflictableType === 'team') {
        this.issueInstitutionalConflictForTeam(false)
      }
    },
    setConflicts: function (
      id, hoverOrPanel, clashOrHistory,
      eventType, conflictType, state, issuerType
    ) {
      // Receive a conflict message from elsewhere and activate the conflict
      if (id !== this.conflictable.id) {
        return // Conflict doesn't match what is broadcast
      }
      if (this.verboseDebugMode) {
        this.debugLog(
          'setConflicts() ', 2, id, hoverOrPanel, clashOrHistory,
          eventType, conflictType, state, issuerType
        )
      }
      if (clashOrHistory === 'histories') {
        conflictType = 'histories' // Histories have seperate type; override it
        const currentHistory = this.isConflicted[hoverOrPanel][conflictType]
        if (currentHistory && state >= currentHistory) {
          return // We only want the most recent history 'seen' state to show
        }
      }
      this.isConflicted[hoverOrPanel][conflictType] = state
    },
    setInstitutionConflicts: function (
      id, hoverOrPanel, clashOrHistory,
      eventType, conflictType, state, issuerType
    ) {
      if (this.isHovering === true) {
        return // Don't show institutional conflicts on self (when hovering)
      }
      if (issuerType === 'team' && this.conflictableType === 'team') {
        return // Teams can't conflict others (only applies to institutionals)
      }
      if (this.verboseDebugMode) {
        this.debugLog(
          'setInstitutionConflicts()', 2, this.conflictable.id,
          hoverOrPanel, clashOrHistory, this.conflictableType,
          conflictType, state, issuerType
        )
      }
      // Check if the supplied institutional ID matches any of this object's clashes
      if (this.conflictingInstitutionIDs.indexOf(id) !== -1) {
        this.isConflicted[hoverOrPanel].institution = state
      }
    },
    resetConflicts: function (hoverOrPanel) {
      this.isConflicted[hoverOrPanel].team = false
      this.isConflicted[hoverOrPanel].histories = false
      this.isConflicted[hoverOrPanel].adjudicator = false
      this.isConflicted[hoverOrPanel].institution = false
    },
  },
}
</script>
