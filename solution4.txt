clear; clc;

%% ================= Basic Parameters =================
teams = {
    'Atlanta Dream','Georgia'
    'Chicago Sky','Illinois'
    'Connecticut Sun','Connecticut'
    'Indiana Fever','Indiana'
    'New York Liberty','Newyork'
    'Washington Mystics','Districtofcolumbia'
    'Dallas Wings','Texas'
    'Las Vegas Aces','Nevada'
    'Los Angeles Sparks','California'
    'Minnesota Lynx','Minnesota'
    'Phoenix Mercury','Arizona'
    'Seattle Storm','Washington'
};

year = 2025;
cap_2024_base = 1463200;
cap_min_ratio = 0.837;
growth_rate = 1.03;

% League expansion parameters
num_teams_old = 12;
num_teams_new = 12;
expansion_ratio = num_teams_new / num_teams_old;

growth_multiplier = growth_rate^(year - 2024);
cap_current_base = cap_2024_base * growth_multiplier;
cap_current = cap_current_base * expansion_ratio;
cap_min_current = cap_current * cap_min_ratio;

fprintf('=== %d WNBA Expansion Roster Optimization ===\n', year);
fprintf('Number of teams after expansion: %d (Original: %d)\n', num_teams_new, num_teams_old);
fprintf('Adjusted salary cap: %.2f USD\n', cap_current);
fprintf('Minimum salary: %.2f USD\n', cap_min_current);

%% ================= Read Data =================
filename = 'Constraints and Objective Function.xlsx';
if exist(filename, 'file') ~= 2
    error('File %s not found, please verify the filename or path.', filename);
end

T = readtable(filename);
n = height(T);

% Convert salary data
salary_cell = T.Salary2024;
salary = zeros(n, 1);
for i = 1:n
    if iscell(salary_cell)
        raw = salary_cell{i};
    else
        raw = salary_cell(i);
    end
    if iscell(raw)
        str = raw{1};
    elseif isnumeric(raw)
        salary(i) = raw;
        continue;
    else
        str = string(raw);
    end
    str_clean = regexprep(str, '[\$,\s]', '');
    val = str2double(str_clean);
    if isnan(val), val = 0; end
    salary(i) = val;
end

% Read PerformanceValue
if ismember('PerformanceValue', T.Properties.VariableNames)
    perf_cell = T.PerformanceValue;
    perf = zeros(n, 1);
    for i = 1:n
        if iscell(perf_cell)
            raw = perf_cell{i};
        else
            raw = perf_cell(i);
        end
        if iscell(raw)
            str = raw{1};
        elseif isnumeric(raw)
            perf(i) = raw;
            continue;
        else
            str = string(raw);
        end
        val = str2double(str);
        if isnan(val), val = 0; end
        perf(i) = val;
    end
else
    error('Column PerformanceValue not found, please check the Excel file.');
end

sign   = T.Signing2024;
pos    = T.Position;
team24 = T.Team2024;
state24 = T.Team_State;
names  = T.Name;

% Identify rookies (N1-N24)
is_rookie = false(n, 1);
for i = 1:n
    if startsWith(names{i}, 'N')
        num = str2double(strrep(names{i}, 'N', ''));
        if ~isnan(num) && num >= 1 && num <= 24
            is_rookie(i) = true;
        end
    end
end

%% ================= Result Storage =================
result_all = cell(size(teams,1), 22);

%% ================= Main Loop (Team-by-Team Optimization) =================
for t = 1:size(teams,1)
    team_name  = teams{t,1};
    team_state = teams{t,2};
    fprintf('\nOptimizing team: %s ...\n', team_name);

    % Define player pool I for this team
    is_common_tradable = ismember(sign, {'UFA','RFA','Rookie','Reserved','Hardship','--'});
    is_my_mandatory = (ismember(sign, {'Core','SuspCE'}) & strcmp(team24, team_name));
    I = find(is_common_tradable | is_my_mandatory);
    m = length(I);

    if m < 12
        warning('Team %s has insufficient eligible players (%d), cannot proceed.', team_name, m);
        continue;
    end

    % Prepare data
    target_col = strcat(team_state, '_CommercialValueScore');
    if ~ismember(target_col, T.Properties.VariableNames)
        warning('Column %s not found, skipping team %s', target_col, team_name);
        continue;
    end
    value_team = T.(target_col);

    pop_team = value_team - perf;

    % Objective function: Max sum(CommercialValue) -> f = -value_team
    f = -value_team(I);
    intcon = 1:m;

    % Build constraints
    A = []; b = [];
    Aeq = []; beq = [];

    % (P1.1) Team size = 12
    Aeq = [Aeq; ones(1,m)];
    beq = [beq; 12];

    % (P1.2) Salary cap & minimum salary
    A = [A; salary(I)'];
    b = [b; cap_current];
    A = [A; -salary(I)'];
    b = [b; -cap_min_current];

    % (P1.3) Retain mandatory players (Core/SuspCE)
    my_mandatory_indices_in_T = find(is_my_mandatory);
    count_mandatory = 0;

    for k = 1:length(my_mandatory_indices_in_T)
        glob_idx = my_mandatory_indices_in_T(k);
        loc_idx = find(I == glob_idx);
        if ~isempty(loc_idx)
            row = zeros(1,m);
            row(loc_idx) = 1;
            Aeq = [Aeq; row];
            beq = [beq; 1];
            count_mandatory = count_mandatory + 1;
        end
    end
    fprintf('   Mandatory Core/SuspCE players retained: %d\n', count_mandatory);

    % (P1.4) Roster continuity adjustment - constraint on new player acquisitions
    current_team_indices = find(strcmp(team24, team_name));
    is_new_player = ~ismember(I, current_team_indices);

    if sum(is_new_player) > 0
        A = [A; double(is_new_player)'];
        b = [b; 4];
        fprintf('   Constraint added: Number of new players (rookies/FA) <= 4\n');
    end

    % (P1.5) Total rookie constraint
    rookie_in_I = is_rookie(I);
    if sum(rookie_in_I) > 0
        A = [A; double(rookie_in_I)'];
        b = [b; 2];
    end

    % Secondary constraints (position balance)
    A_p2 = []; b_p2 = [];
    isG = double(contains(pos(I), 'G'));
    isF = double(contains(pos(I), 'F'));
    isC = double(contains(pos(I), 'C'));

    A_p2 = [A_p2; -isG'; isG']; b_p2 = [b_p2; -3; 5];
    A_p2 = [A_p2; -isF'; isF']; b_p2 = [b_p2; -3; 5];
    A_p2 = [A_p2; -isC'; isC']; b_p2 = [b_p2; -2; 4];

    % Solve
    lb = zeros(m,1);
    ub = ones(m,1);
    options = optimoptions('intlinprog','Display','off');

    fprintf('   Solving...\n');
    [x, ~, exitflag] = intlinprog(f, intcon, [A; A_p2], [b; b_p2], Aeq, beq, lb, ub, options);

    if exitflag <= 0
        warning('   Position constraints cannot be satisfied, attempting with hard constraints only...');
        [x, ~, exitflag] = intlinprog(f, intcon, A, b, Aeq, beq, lb, ub, options);
    end

    % Process results
    if exitflag > 0
        chosen_indices = I(x > 0.5);

        % Calculate statistics
        target_col_2024 = strcat(team_state, '_CommercialValueScore');
        if ismember(target_col_2024, T.Properties.VariableNames)
            current_roster = find(strcmp(team24, team_name));
            val24 = sum(T.(target_col_2024)(current_roster));
            pop24 = sum(pop_team(current_roster));
        else
            val24 = 0;
            pop24 = 0;
        end

        val25 = sum(value_team(chosen_indices));
        pop25 = sum(pop_team(chosen_indices));

        tot_sal = sum(salary(chosen_indices));

        % Decision logic
        decision = '';
        if pop25 > pop24
            decision = 'Increase media promotion; recommend more interviews for high-popularity players';
        else
            decision = 'No need for additional media promotion';
        end

        % Sort by Pop Value (pop_team) in descending order
        [~, sort_order] = sort(pop_team(chosen_indices), 'descend');
        sorted_indices = chosen_indices(sort_order);

        % Fill result array
        result_all{t,1} = team_name;
        result_all{t,2} = val24;
        result_all{t,3} = val25;
        result_all{t,4} = val25 - val24;
        result_all{t,5} = tot_sal;
        result_all{t,6} = pop24;
        result_all{t,7} = pop25;
        result_all{t,8} = decision;

        for k = 1:14
            col_idx = 8 + k;
            if k <= length(sorted_indices)
                p_idx = sorted_indices(k);
                p_name = names{p_idx};
                p_val  = pop_team(p_idx);
                result_all{t,col_idx} = sprintf('%s (%.4f)', p_name, p_val);
            else
                result_all{t,col_idx} = '';
            end
        end
        fprintf('   Optimization successful! Value increase: %.2f, Decision: %s\n', val25 - val24, decision);
    else
        fprintf('   No solution found.\n');
        result_all{t,1} = team_name;
        result_all{t,2} = 0;
        result_all{t,3} = NaN;
        result_all{t,4} = NaN;
        result_all{t,5} = NaN;
        result_all{t,6} = NaN;
        result_all{t,7} = NaN;
        result_all{t,8} = '';
    end
end

%% ================= Output Excel =================
varNames = [
    {'Team','Value2024','Value2025','Diff','TotalSalary','Pop2024','Pop2025','Decision'}, ...
    arrayfun(@(x)sprintf('Player%d',x),1:14,'UniformOutput',false)
];
result_table = cell2table(result_all,'VariableNames',varNames);
outfile = 'Question4_MediaPromotionPopularityValue_Revised.xlsx';
writetable(result_table, outfile);

fprintf('\nAll results saved to: %s\n', outfile);