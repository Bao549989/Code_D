clear; clc;

%% ================= Auxiliary Function: Generate Position Markers =================
function player_name_with_mark = add_position_mark(player_name, position, is_starter)
    % Generate markers based on position and starter status
    % Starter markers: SG (Starter Guard), SF (Starter Forward), SC (Starter Center)
    % Non-starter markers: NG (Non-starter Guard), NF (Non-starter Forward), NC (Non-starter Center)
    
    position = string(position);
    
    % Determine position type
    has_G = contains(position, 'G');
    has_F = contains(position, 'F'); 
    has_C = contains(position, 'C');
    
    if is_starter
        mark_G = 'SG';
        mark_F = 'SF';
        mark_C = 'SC';
    else
        mark_G = 'NG';
        mark_F = 'NF';
        mark_C = 'NC';
    end
    
    marks = {};
    if has_G, marks{end+1} = mark_G; end
    if has_F, marks{end+1} = mark_F; end
    if has_C, marks{end+1} = mark_C; end
    
    if isempty(marks)
        player_name_with_mark = player_name;
    else
        mark_str = strjoin(marks, '-');
        player_name_with_mark = sprintf('%s(%s)', player_name, mark_str);
    end
end

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
num_teams_new = 12; % Assume expansion to 12 teams
expansion_ratio = num_teams_new / num_teams_old;

% Dynamically calculate current salary cap and minimum salary (considering expansion)
growth_multiplier = growth_rate^(year - 2024);
cap_current_base = cap_2024_base * growth_multiplier;
cap_current = cap_current_base * expansion_ratio; % Increase salary cap
cap_min_current = cap_current * cap_min_ratio;

fprintf('=== WNBA Expansion Roster Optimization %d ===\n', year);
fprintf('Number of teams after expansion: %d (original %d)\n', num_teams_new, num_teams_old);
fprintf('Adjusted salary cap: %.2f USD\n', cap_current);
fprintf('Minimum salary: %.2f USD\n', cap_min_current);

%% ================= Global Constants =================
league_avg_age = 28.5;           % League average age
total_games = 40;                % Total regular season games
min_game_ratio = 0.80;           % Minimum game participation ratio for starters
min_ws_ratio = 0.50;             % Minimum WS ratio of non-starters to starters

%% ================= Data Reading =================
filename = 'Constraints and Objective Function.xlsx';
if exist(filename, 'file') ~= 2
    error('File %s does not exist. Please check the file name or path.', filename);
end

T = readtable(filename);
n = height(T);

% Salary data conversion
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
    if isnan(val)
        salary(i) = 0;
    else
        salary(i) = val;
    end
end

% Read player game appearances (column G)
if ~ismember('G', T.Properties.VariableNames)
    error('The table must contain a column named G for storing player game appearances');
end
games_played = T.G;
games_played(isnan(games_played)) = 0;

% Other fields
sign   = T.Signing2024;
pos    = T.Position;
team24 = T.Team2024;
state24 = T.Team_State;
names  = T.Name;
ws_values   = T.WS;             % Win Shares (WS)
age_2024    = T.Age2024;        % Age in 2024
age_current = age_2024 + (year - 2024);  % Actual age in current year
age_current(isnan(age_current)) = league_avg_age; % Fill missing values with league average age

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

%% ================= Result Storage Initialization =================
result_all = cell(size(teams,1), 22); % Corrected to 22 columns (10 basic columns + 12 player columns)

%% ================= Main Loop: Team-by-Team Optimization =================
for t = 1:size(teams,1)
    team_name  = teams{t,1};
    team_state = teams{t,2};
    fprintf('\n========== Optimizing Team: %s ==========\n', team_name);

    % --- 1. Define player pool I for the team ---
    is_common_tradable = ismember(sign, {'UFA','RFA','Rookie','Reserved','Hardship','--'});
    is_my_mandatory = (ismember(sign, {'Core','SuspCE'}) & strcmp(team24, team_name));
    I = find(is_common_tradable | is_my_mandatory);
    m = length(I);
    
    if m < 12
        warning('Team %s has insufficient optional players (%d players), optimization skipped.', team_name, m);
        continue;
    end
    
    % --- 2. Prepare data ---
    target_col = strcat(team_state, '_CommercialValueScore');
    if ~ismember(target_col, T.Properties.VariableNames)
        warning('Column %s not found, skipping team %s', target_col, team_name);
        continue;
    end
    value_team = T.(target_col);
    
    % Objective function: Max sum(value)
    f = -value_team(I);
    intcon = 1:m;

    % --- 3. Build Priority 1 Constraints (Hard Constraints) ---
    A_hard = []; b_hard = [];      % Inequality constraints
    Aeq_hard = []; beq_hard = [];  % Equality constraints

    % (P1.1) Team size = 12 players
    Aeq_hard = [Aeq_hard; ones(1,m)];
    beq_hard = [beq_hard; 12];

    % (P1.2) Salary cap & minimum salary
    A_hard = [A_hard; salary(I)'];         
    b_hard = [b_hard; cap_current];
    A_hard = [A_hard; -salary(I)'];        
    b_hard = [b_hard; -cap_min_current];

    % (P1.3) Mandatory retention of Core/SuspCE players
    my_mandatory_indices_in_T = find(is_my_mandatory);
    count_mandatory = 0;
    
    for k = 1:length(my_mandatory_indices_in_T)
        glob_idx = my_mandatory_indices_in_T(k);
        loc_idx = find(I == glob_idx);
        if ~isempty(loc_idx)
            row = zeros(1,m);
            row(loc_idx) = 1;
            Aeq_hard = [Aeq_hard; row];
            beq_hard = [beq_hard; 1];
            count_mandatory = count_mandatory + 1;
        end
    end
    fprintf('  Mandatory retained Core/SuspCE players: %d\n', count_mandatory);

    % (P1.4) Maximum 4 newly recruited players
    current_team_indices = find(strcmp(team24, team_name));
    is_new_player = ~ismember(I, current_team_indices);
    
    if sum(is_new_player) > 0
        A_hard = [A_hard; double(is_new_player)'];
        b_hard = [b_hard; 4];
    end
    
    % (P1.5) Rookie count constraint (maximum 2 rookies)
    rookie_in_I = is_rookie(I);
    if sum(rookie_in_I) > 0
        A_hard = [A_hard; double(rookie_in_I)'];
        b_hard = [b_hard; 2];
    end

    % --- 4. Build Priority 2 Constraints (3 New Constraints) ---
    A_second = []; b_second = [];
    
    % Extract necessary data
    pos_I = pos(I);
    ws_I = ws_values(I);
    games_I = games_played(I);
    age_I = age_current(I);
    
    % Priority 2 Constraint 1: Team average age ≤ league average age (28.5)
    % Transformed to: sum(age_i * x_i) ≤ 28.5 * 12
    age_sum_limit = league_avg_age * 12;
    A_second = [A_second; age_I'];
    b_second = [b_second; age_sum_limit];
    
    % Priority 2 Constraints 2 and 3 require knowing starters for each position
    % Since starters are part of optimization variables, linear approximation is used
    % Conservative approach: assume the player with highest WS in each position is the starter candidate
    
    % Position identifiers
    isG = double(contains(pos(I), 'G'));
    isF = double(contains(pos(I), 'F'));
    isC = double(contains(pos(I), 'C'));
    
    % Group by position
    G_indices = find(isG);
    F_indices = find(isF);
    C_indices = find(isC);
    
    % Add constraints for each position
    for pos_type = {'G', 'F', 'C'}
        if strcmp(pos_type, 'G')
            pos_indices = G_indices;
        elseif strcmp(pos_type, 'F')
            pos_indices = F_indices;
        else
            pos_indices = C_indices;
        end
        
        if length(pos_indices) >= 2
            % Find the player with highest WS in the position (starter candidate)
            [~, ws_order] = sort(ws_I(pos_indices), 'descend');
            main_player_idx = pos_indices(ws_order(1));
            
            % Constraint 2: Starter's game appearances ≥ 70% of total games
            min_games = min_game_ratio * total_games;
            row_main_games = zeros(1, m);
            row_main_games(main_player_idx) = -games_I(main_player_idx);
            A_second = [A_second; row_main_games];
            b_second = [b_second; -min_games];
            
            % Constraint 3: At least one non-starter in the position has WS ≥ 50% of starter's WS
            if length(pos_indices) >= 2
                main_ws = ws_I(main_player_idx);
                min_ws = min_ws_ratio * main_ws;
                
                % Create variables for non-starter players
                non_main_indices = pos_indices(ws_order(2:end));
                
                % Constraint: Select at least one non-starter with WS ≥ 50% of starter's WS
                % This is a combinatorial constraint, linear approximation is used
                for non_idx = non_main_indices
                    if ws_I(non_idx) >= min_ws
                        % Add constraint to ensure at least one such non-starter is selected
                        row_non_main = zeros(1, m);
                        row_non_main(non_idx) = 1;
                        A_second = [A_second; -row_non_main];
                        b_second = [b_second; -1];
                        break; % Only one such constraint is needed
                    end
                end
            end
        end
    end

    % --- 5. Build Priority 3 Constraints (Position Balance) ---
    A_third = []; b_third = [];

    % Position quantity constraints
    A_third = [A_third; -isG'; isG']; b_third = [b_third; -3; 5];
    A_third = [A_third; -isF'; isF']; b_third = [b_third; -3; 5];
    A_third = [A_third; -isC'; isC']; b_third = [b_third; -2; 4];

    % --- 6. Hierarchical Solution ---
    lb = zeros(m,1);
    ub = ones(m,1);
    options = optimoptions('intlinprog','Display','off');
    
    fprintf('  Solving hierarchically...\n');
    
    % Level 1: Priority 1 + Priority 2 + Priority 3
    A_all = [A_hard; A_second; A_third];
    b_all = [b_hard; b_second; b_third];
    
    [x, ~, exitflag] = intlinprog(f, intcon, A_all, b_all, Aeq_hard, beq_hard, lb, ub, options);
    
    % Level 2: Priority 1 + Priority 2 (abandon position constraints)
    if exitflag <= 0
        fprintf('  Level 1 has no solution, trying Level 2 (abandon position constraints)...\n');
        A_all = [A_hard; A_second];
        b_all = [b_hard; b_second];
        [x, ~, exitflag] = intlinprog(f, intcon, A_all, b_all, Aeq_hard, beq_hard, lb, ub, options);
    end
    
    % Level 3: Only Priority 1 (abandon Priority 2 and 3 constraints)
    if exitflag <= 0
        fprintf('  Level 2 has no solution, trying Level 3 (only hard constraints)...\n');
        A_all = A_hard;
        b_all = b_hard;
        [x, ~, exitflag] = intlinprog(f, intcon, A_all, b_all, Aeq_hard, beq_hard, lb, ub, options);
    end

    % --- 7. Result Processing ---
    if exitflag > 0
        chosen_indices = I(x > 0.5);
        
        % Statistical data
        target_col_2024 = strcat(team_state, '_CommercialValueScore');
        if ismember(target_col_2024, T.Properties.VariableNames)
             current_roster = find(strcmp(team24, team_name));
             val24 = sum(T.(target_col_2024)(current_roster));
        else
             val24 = 0;
        end
        val25 = sum(value_team(chosen_indices));
        tot_sal = sum(salary(chosen_indices));
        
        % Calculate average age
        avg_age = mean(age_current(chosen_indices));
        
        % Check compliance with Priority 2 constraints
        constraint_satisfied = check_new_constraints(chosen_indices, I, pos_I, ws_I, games_I, age_I, ...
            league_avg_age, total_games, min_game_ratio, min_ws_ratio);
        
        % ========== New: Add starter markers to players ==========
        % Determine starters (highest WS players) for each position
        chosen_positions = pos(chosen_indices);
        chosen_ws = ws_values(chosen_indices);
        chosen_names = names(chosen_indices);
        
        % Identify starters by position group
        starter_indices = [];
        pos_groups = {'G', 'F', 'C'};
        
        for p = 1:length(pos_groups)
            pos_type = pos_groups{p};
            % Find all players in the position
            pos_players = find(contains(chosen_positions, pos_type));
            if ~isempty(pos_players)
                % Sort by WS, select top as starters
                [~, ws_order] = sort(chosen_ws(pos_players), 'descend');
                % Select at least one starter, maximum 2 per position
                n_starters = min(2, length(pos_players)); % Max 2 starters per position
                starters = pos_players(ws_order(1:n_starters));
                starter_indices = [starter_indices; starters];
            end
        end
        
        % Generate player names with markers
        marked_names = cell(length(chosen_indices), 1);
        for i = 1:length(chosen_indices)
            is_starter = ismember(i, starter_indices);
            marked_names{i} = add_position_mark(chosen_names{i}, chosen_positions{i}, is_starter);
        end
        % ==========================================================
        
        result_all{t,1} = team_name;
        result_all{t,2} = val24;
        result_all{t,3} = val25;
        result_all{t,4} = val25 - val24;
        result_all{t,5} = tot_sal;
        result_all{t,6} = avg_age;
        result_all{t,7} = constraint_satisfied.age_constraint;
        result_all{t,8} = constraint_satisfied.games_constraint;
        result_all{t,9} = constraint_satisfied.ws_constraint;
        
        % Record solution level (corrected column index: 10th column)
        if isempty(A_third) || all(A_third*x <= b_third)
            result_all{t,10} = 'All Constraints';
        elseif isempty(A_second) || all(A_second*x <= b_second)
            result_all{t,10} = 'Priority 1 + Priority 2';
        else
            result_all{t,10} = 'Only Hard Constraints';
        end
        
        % Assign 12 players (columns 11 ~ 22)
        for k = 1:12
            col_idx = 10 + k; % First 10 columns are basic info, players start from column 11
            if k <= length(marked_names)  % Use names with markers
                result_all{t,col_idx} = marked_names{k};
            else
                result_all{t,col_idx} = '';
            end
        end
        
        fprintf('  Optimization successful! Value increase: %.2f, Average age: %.2f\n', val25 - val24, avg_age);
        fprintf('  Constraint compliance: Age:%d, Games:%d, WS:%d\n', ...
            constraint_satisfied.age_constraint, constraint_satisfied.games_constraint, constraint_satisfied.ws_constraint);
    else
        % Processing for no solution
        fprintf('  No solution available.\n');
        result_all{t,1} = team_name;
        result_all{t,2} = 0;
        result_all{t,3} = NaN;
        result_all{t,4} = NaN;
        result_all{t,5} = NaN;
        result_all{t,6} = NaN;
        result_all{t,7} = 0;
        result_all{t,8} = 0;
        result_all{t,9} = 0;
        result_all{t,10} = 'No Solution';
        % Fill player columns with empty values (columns 11 ~ 22)
        for k = 11:22
            result_all{t,k} = '';
        end
    end
end

%% ================= Export to Excel =================
varNames = [
    {'Team','Value2024','Value2025','Diff','TotalSalary','AvgAge',...
     'AgeConstraint','GamesConstraint','WSConstraint','SolutionLevel'}, ...
    arrayfun(@(x)sprintf('Player%d',x),1:12,'UniformOutput',false)
];
result_table = cell2table(result_all,'VariableNames',varNames);
outfile = 'WNBA_Optimization_With_New_Priorities.xlsx';
writetable(result_table, outfile);

fprintf('\nAll results saved to: %s\n', outfile);

%% ================= Auxiliary Function: Check New Constraints =================
function constraint_satisfied = check_new_constraints(chosen_indices, I, pos_I, ws_I, games_I, age_I, ...
    league_avg_age, total_games, min_game_ratio, min_ws_ratio)
    
    % Find indices of selected players in pool I
    [~, chosen_in_I] = intersect(I, chosen_indices);
    
    % Initialize results
    constraint_satisfied.age_constraint = 1;
    constraint_satisfied.games_constraint = 1;
    constraint_satisfied.ws_constraint = 1;
    
    % Check age constraint
    avg_age = mean(age_I(chosen_in_I));
    if avg_age > league_avg_age
        constraint_satisfied.age_constraint = 0;
    end
    
    % Check by position group
    pos_types = {'G', 'F', 'C'};
    
    for p = 1:length(pos_types)
        pos_type = pos_types{p};
        
        % Find selected players in the position
        pos_players = chosen_in_I(strcmp(pos_I(chosen_in_I), pos_type));
        
        if isempty(pos_players)
            continue;
        end
        
        % Find the player with highest WS in the position (starter)
        [~, ws_order] = sort(ws_I(pos_players), 'descend');
        main_player = pos_players(ws_order(1));
        
        % Check starter game appearance constraint
        if games_I(main_player) < min_game_ratio * total_games
            constraint_satisfied.games_constraint = 0;
        end
        
        % Check non-starter WS constraint
        if length(pos_players) >= 2
            main_ws = ws_I(main_player);
            min_ws = min_ws_ratio * main_ws;
            
            % Check if any non-starter has WS ≥ min_ws
            non_main_players = pos_players(ws_order(2:end));
            has_valid_backup = false;
            
            for i = 1:length(non_main_players)
                if ws_I(non_main_players(i)) >= min_ws
                    has_valid_backup = true;
                    break;
                end
            end
            
            if ~has_valid_backup
                constraint_satisfied.ws_constraint = 0;
            end
        end
    end
end