.. _pydfs-lineup-optimizer-usage:


Usage
=====

Base Usage
----------
Creating optimal lineups with **pydfs-lineup-optimizer** is very simple.
Firstly you should create optimizer. You can do this using
shortcut get_optimizer. You must provide daily fantasy site for it and kind of sport.
If site doesn't support specified sport you will get NotImplementedError.

.. code-block:: python

    from pydfs_lineup_optimizer import get_optimizer, Site, Sport


    optimizer = get_optimizer(Site.FANDUEL, Sport.BASKETBALL)

After that you need to load players into your optimizer. You have 2 options:
First is to load players from CSV file like this:

.. code-block:: python

    optimizer.load_players_from_csv("path_to_csv")

.. note::

   CSV file must have the same format as export file in specified dfs site, if you have custom CSV file this method will not work.
   Also this method raises `NotImplementedError` for FanBall site because it hasn't export csv feature.

Or you can load players using load_players method and pass list with players.

.. code-block:: python

    from pydfs_lineup_optimizer import Player
    optimizer.load_players(players)  # players is list of Player objects

After player loading you can create your optimal lineups with following code:

.. code-block:: python

    lineups = optimizer.optimize(n=10)

Where n is a number of lineups that you want generate.

.. note::

   Optimize method returns generator instead of list.

Example of base usage
---------------------

Below is a full example of how **pydfs-lineup-optimizer** can be used to generate optimal lineups.

.. code-block:: python

    optimizer = get_optimizer(Site.YAHOO, Sport.BASKETBALL)
    optimizer.load_players_from_csv("yahoo-NBA.csv")
    for lineup in optimizer.optimize(n=10):
        print(lineup)
        print(lineup.players)  # list of players
        print(lineup.fantasy_points_projection)
        print(lineup.salary_costs)

Advanced Usage
--------------

For generating optimal lineups you may need to lock some players that you want to see in your lineup.
You can do this using following code:

.. code-block:: python

    player = optimizer.get_player_by_name('Rodney Hood') # find player with specified name in your optimizer
    second_player = optimizer.get_player_by_id('ID00001')  # find player with player id
    optimizer.add_player_to_lineup(player) # lock this player in lineup
    optimizer.add_player_to_lineup(second_player)

Locked players can be unlocked as well:

.. code-block:: python

    optimizer.remove_player_from_lineup(player)

Also you can exclude some players from optimization process and restore players as well:

.. code-block:: python

    optimizer.remove_player(player)
    optimizer.restore_player(player)

You can specify maximum and minimum exposures for some players or max exposure for all players, you have several ways how to do this.
You can add "Max Exposure" and "Min Exposure" columns with exposure percentage for some players to csv that will be parsed while players loading.
Or you can set max_exposure/min_exposure property in Player object. If you want to set fixed max exposure for all players you can
pass max_exposure parameter to optimize method

.. code-block:: python

    player = optimizer.players[0]  # get random player from optimizer players
    player.max_exposure = 0.5  # set 50% max exposure
    player.min_exposure = 0.3  # set 30% min exposure

    lineups = optimizer.optimize(n=10, max_exposure=0.3)  # set 30% exposure for all players

.. note::

   Exposure working with locked players, so if you lock some player and set max exposure to 50% percentage
   this player will appears only in 50% lineups.
   Player max exposure has higher priority than max_exposure passed in optimize method.
   Exposure percentage rounds to ceil.

By default, the optimizer generates lineups based on the total number of lineups. It means if you have a player with a
huge projection it will be selected only in first n lineups.
You can change this behavior to another algorithm where exposure calculates
after each generated lineup. For example, if you have a player with a huge projection and
set his max_exposure to 0.5 optimizer will select him in the first lineup then skip 2 lineups with this player
(because he has 100% exposure after the first lineup and 50% after the second lineup that is not less than specified value)
and will add this player to the fourth lineup. In this case, lineups can be unordered.

.. code-block:: python

    from pydfs_lineup_optimizer import AfterEachExposureStrategy

    lineups = optimizer.optimize(n=10, max_exposure=0.3, exposure_strategy=AfterEachExposureStrategy)

After optimization you can print to console list with statistic about players used in lineups.

.. code-block::

    optimizer.print_statistic()

Example of advanced usage
-------------------------

Below is an full example of how **pydfs-lineup-optimizer** can be used to generate optimal lineups with user constraints.

.. code-block:: python

    optimizer = get_optimizer(Site.YAHOO, Sport.BASKETBALL)
    optimizer.load_players_from_csv("yahoo-NBA.csv")
    nets_centers = filter(lambda p: p.team == 'Nets' and 'C' in p.positions, optimizer.players)
    for player in nets_centers:
        optimizer.remove_player(player)  # Remove all Nets centers from optimizer
    harden = optimizer.get_player_by_name('Harden')
    westbrook = optimizer.get_player_by_name('Westbrook')  # Get Harden and Westbrook
    harden.max_exposure = 0.6
    westbrook.max_exposure = 0.4  # Set exposures for Harden and Westbrook
    optimizer.add_player_to_lineup(harden)
    optimizer.add_player_to_lineup(westbrook)  # Lock Harden and Westbrook
    for lineup in optimizer.optimize(n=10, max_exposure=0.3):
        print(lineup)

Late-Swap
--------------------

Optimizer provides additional functionality that allows to re-optimize existed lineups.
Currently this feature implemented for DK and FanDuel.
For this you should load lineups, you can do it from csv file generated for specific contest.
Then you should pass loaded lineups to `optimize_lineups` method.
Players with started game will be locked on specific positions and optimizer will change only players with upcoming game.

.. code-block:: python

    csv_filename = "dk_nba.csv"
    optimizer = get_optimizer(Site.DRAFTKINGS, Sport.BASKETBALL)
    optimizer.load_players_from_csv(csv_filename)
    lineups = optimizer.load_lineups_from_csv(csv_filename)
    for lineup in optimizer.optimize_lineups(lineups):
        print(lineup)

Because FanDuel doesn't provide information about locked player and games start time you should manually add information about started games like in example below:

.. code-block:: python

    csv_filename = "fd_nba.csv"
    optimizer = get_optimizer(Site.FANDUEL, Sport.BASKETBALL)
    optimizer.load_players_from_csv(csv_filename)
    lineups = optimizer.load_lineups_from_csv(csv_filename)
    locked_teams = {'DET', 'MIA', 'BOS', 'NYK'}
    for game in optimizer.games:
        if game.home_team in locked_teams or game.away_team in locked_teams:
            game.game_started = True
    for lineup in optimizer.optimize_lineups(lineups):
        print(lineup)

For parsing dates of games for DK library uses US/Eastern timezone by default.
You can change it using `set_timezone` function:

.. code-block:: python

    from pydfs_lineup_optimizer import set_timezone

    set_timezone('UTC')

Export lineups
==============

You can export lineups into a csv file. For this you should call export method of the optimizer after you generate all lineups.

.. code-block:: python

    from pydfs_lineup_optimizer import get_optimizer, Site, Sport, CSVLineupExporter

    optimizer = get_optimizer(Site.DRAFTKINGS, Sport.BASKETBALL)
    optimizer.load_players_from_csv("players.csv")

    # if you want to see lineups on screen
    for lineup in optimizer.optimize(10):
        print(lineup)
    optimizer.export('result.csv')

    # if you don't need to see lineups on screen
    lineups = list(optimizer.optimize(10))
    optimizer.export('result.csv')

Adjusting player fantasy points
===============================

By default optimizer uses value of `fppg` property of player for optimizing.
You can change this behaviour by providing a custom fantasy points strategy using `set_fantasy_points_strategy` method.
There are several strategies already implemented in this package:

- RandomFantasyPointsStrategy
- ProgressiveFantasyPointsStrategy

RandomFantasyPointsStrategy adds some deviation for players projection for creating less optimized but more randomized lineups.
You can set this deviation when creating strategy by default min deviation is 0 and max deviation is 12%.
You also can specify player specific deviation using `min_deviation` and `max_deviation` attributes for player,
or using additional columns `Min Deviation` and `Max Deviation` in import csv.
Also you can randomize players fppg by specifying projection range using `fppg_floor` and `fppg_ceil` attributes for player or
`Projection Floor` and `Projection Ceil` csv columns. In this case this method has priority over deviation.
It works only if both fields are specified.

.. code-block:: python

    optimizer.set_fantasy_points_strategy(RandomFantasyPointsStrategy(max_deviation=0.2))  # set random strategy with custom max_deviation
    harden = optimizer.get_player_by_name('Harden')
    harden.min_deviation = 0.3
    harden.max_deviation = 0.6  # Set different deviation for player
    westbrook = optimizer.get_player_by_name('Westbrook')
    westbrook.min_deviation = 0  # Disable randomness for this player
    westbrook.max_deviation = 0
    doncic = optimizer.get_player_by_name('Doncic')
    doncic.fppg_floor = 60  # Randomize using projection range
    doncic.fppg_ceil = 90
    lineups = optimizer.optimize(n=10)

.. note::

    With RandomFantasyPointsStrategy optimizer generate lineups without ordering by max points projection.

ProgressiveFantasyPointsStrategy is another method to randomize optimizer result.
It increases fantasy points for each player that wasn't used in the previous lineup by some specified percent of original fantasy points.
It works cumulatively so fantasy points will be greater if player didn't used in lineup multiple times.
After player will be selected to lineup his points will be reset to the original value.
You can change this value for specific player by setting `progressive_scale` property of Player or by adding `Progressive Scale` column to import csv.

.. code-block:: python

    optimizer.set_fantasy_points_strategy(ProgressiveFantasyPointsStrategy(0.01))  # Set progressive strategy that increase player points by 1%
    optimizer.get_player_by_name('Stephen Curry').progressive_scale = 0.02  # For curry points will be increased by 2%
