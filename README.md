#include <ncurses.h>
#include <iostream>
#include <vector>
#include <string>
#include <fstream>
#include <sstream>
#include <dirent.h>
#include <unistd.h>
#include <signal.h>
#include <algorithm>
#include <map>
#include <chrono>
#include <thread>

struct Process {
    int pid;
    std::string name;
    double cpu_percent;
    long mem_kb;
};

long get_total_cpu_time() {
    std::ifstream file("/proc/stat");
    std::string line;
    std::getline(file, line);
    std::istringstream iss(line);
    std::string cpu;
    iss >> cpu;
    long user, nice, system, idle, iowait, irq, softirq, steal, guest, guest_nice;
    iss >> user >> nice >> system >> idle >> iowait >> irq >> softirq >> steal >> guest >> guest_nice;
    return user + nice + system + idle + iowait + irq + softirq + steal + guest + guest_nice;
}

long get_process_cpu_time(int pid) {
    std::ifstream file("/proc/" + std::to_string(pid) + "/stat");
    std::string line;
    if (!std::getline(file, line)) return 0;
    std::istringstream iss(line);
    int pid2;
    std::string comm;
    char state;
    int ppid, pgrp, session, tty_nr, tpgid;
    unsigned flags;
    unsigned long minflt, cminflt, majflt, cmajflt;
    unsigned long utime, stime;
    iss >> pid2 >> comm >> state >> ppid >> pgrp >> session >> tty_nr >> tpgid >> flags >> minflt >> cminflt >> majflt >> cmajflt >> utime >> stime;
    return utime + stime;
}

long get_process_mem(int pid) {
    std::ifstream file("/proc/" + std::to_string(pid) + "/status");
    std::string line;
    while (std::getline(file, line)) {
        if (line.substr(0, 6) == "VmRSS:") {
            std::istringstream iss(line.substr(6));
            long mem;
            std::string unit;
            iss >> mem >> unit;
            return mem; // in kB
        }
    }
    return 0;
}

std::string get_process_name(int pid) {
    std::ifstream file("/proc/" + std::to_string(pid) + "/comm");
    std::string name;
    if (std::getline(file, name)) {
        return name;
    }
    return "";
}

std::vector<int> get_pids() {
    std::vector<int> pids;
    DIR* dir = opendir("/proc");
    if (!dir) return pids;
    struct dirent* entry;
    while ((entry = readdir(dir)) != nullptr) {
        std::string name = entry->d_name;
        if (!name.empty() && name[0] >= '0' && name[0] <= '9') {
            try {
                pids.push_back(std::stoi(name));
            } catch (...) {}
        }
    }
    closedir(dir);
    return pids;
}

long get_total_mem() {
    std::ifstream file("/proc/meminfo");
    std::string line;
    if (std::getline(file, line)) {
        std::istringstream iss(line);
        std::string mem;
        iss >> mem;
        long total;
        iss >> total;
        return total;
    }
    return 0;
}

long get_used_mem() {
    std::ifstream file("/proc/meminfo");
    std::string line;
    long total = 0, available = 0;
    while (std::getline(file, line)) {
        if (line.substr(0, 9) == "MemTotal:") {
            std::istringstream iss(line.substr(9));
            iss >> total;
        } else if (line.substr(0, 12) == "MemAvailable:") {
            std::istringstream iss(line.substr(12));
            iss >> available;
        }
    }
    return total - available;
}

std::string get_load_avg() {
    std::ifstream file("/proc/loadavg");
    std::string load;
    if (std::getline(file, load)) {
        size_t pos = load.find(' ');
        if (pos != std::string::npos) {
            return load.substr(0, pos);
        }
    }
    return "0.00";
}

int main() {
    initscr();
    noecho();
    cbreak();
    keypad(stdscr, TRUE);
    timeout(1000); // 1 second timeout for getch

    std::map<int, long> prev_process_cpu;
    long prev_total_cpu = 0;
    char sort_by = 'c'; // 'c' for CPU, 'm' for memory

    while (true) {
        long total_cpu = get_total_cpu_time();
        std::vector<Process> processes;
        auto pids = get_pids();
        for (int pid : pids) {
            std::string name = get_process_name(pid);
            long mem = get_process_mem(pid);
            long curr_cpu = get_process_cpu_time(pid);
            long prev_cpu = prev_process_cpu[pid];
            double cpu_percent = 0.0;
            if (prev_total_cpu > 0) {
                long delta_cpu = curr_cpu - prev_cpu;
                long delta_total = total_cpu - prev_total_cpu;
                if (delta_total > 0) {
                    cpu_percent = (double)delta_cpu / delta_total * 100.0;
                }
            }
            processes.push_back({pid, name, cpu_percent, mem});
            prev_process_cpu[pid] = curr_cpu;
        }
        prev_total_cpu = total_cpu;

        // Sort processes
        if (sort_by == 'c') {
            std::sort(processes.begin(), processes.end(), [](const Process& a, const Process& b) {
                return a.cpu_percent > b.cpu_percent;
            });
        } else if (sort_by == 'm') {
            std::sort(processes.begin(), processes.end(), [](const Process& a, const Process& b) {
                return a.mem_kb > b.mem_kb;
            });
        }

        // Get system info
        long total_mem = get_total_mem();
        long used_mem = get_used_mem();
        std::string load = get_load_avg();

        // Clear screen
        clear();

        // Print header
        mvprintw(0, 0, "System Monitor (q to quit, c to sort by CPU, m to sort by Mem, k to kill process)");
        mvprintw(1, 0, "Tasks: %d, Load: %s, Mem: %ld/%ld MB", (int)processes.size(), load.c_str(), used_mem / 1024, total_mem / 1024);
        mvprintw(2, 0, "%5s %20s %5s %8s", "PID", "NAME", "CPU%", "MEM(kB)");

        // Print processes (limit to screen height)
        int max_lines = LINES - 4;
        for (int i = 0; i < std::min((int)processes.size(), max_lines); ++i) {
            const auto& p = processes[i];
            mvprintw(3 + i, 0, "%5d %20s %5.1f %8ld", p.pid, p.name.c_str(), p.cpu_percent, p.mem_kb);
        }

        refresh();

        // Handle input
        int ch = getch();
        if (ch == 'q') break;
        else if (ch == 'c') sort_by = 'c';
        else if (ch == 'm') sort_by = 'm';
        else if (ch == 'k') {
            echo();
            mvprintw(LINES - 1, 0, "Enter PID to kill: ");
            char buf[10] = {0};
            getstr(buf);
            int kill_pid = atoi(buf);
            if (kill_pid > 0) {
                kill(kill_pid, SIGKILL);
            }
            noecho();
        }

        // Sleep for 1 second
        std::this_thread::sleep_for(std::chrono::seconds(1));
    }

    endwin();
    return 0;
}
