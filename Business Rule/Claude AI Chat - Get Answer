(function executeRule(current, previous /*null when async*/) {

var claude = new ClaudeAI();
    var result = claude.summarizeIncident(current.short_description + '', current.description + '' );

    var note = result.success ? 'AI Summary:\n' + result.message : 'Claude AI Error: ' + result.error;

    current.setWorkflow(false);     // ← CRITICAL: prevents infinite loop
    current.work_notes = note;
    current.update();

})(current, previous);
