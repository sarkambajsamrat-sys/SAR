#include "Task.h"
#include "TaskExceptions.h"

#include <cstring>
#include <iostream>

const int Task::TM_YEAR_BASE = 1900;
const int Task::END_OF_DAY_HOUR = 23;
const int Task::END_OF_DAY_MINUTE = 59;
const int Task::END_OF_DAY_SECOND = 59;
const int Task::AUTO_DAYLIGHT_SAVING = -1;
const int Task::DATE_BUFFER_SIZE = 32;
const time_t Task::INVALID_TIME = static_cast<time_t>(-1);
const double Task::MAX_HOURS = 10000.0;
const char* const Task::NOT_ASSIGNED_TEXT = "not assigned";
const char* const Task::DATE_FORMAT = "%d.%m.%Y";

// ---------- Private helpers ----------

char* Task::CopyString(const char* const source)
{
    char* copy = new char[strlen(source) + 1];
    strcpy(copy, source);
    return copy;
}

bool Task::IsBlank(const char* const text)
{
    if (text == nullptr)
    {
        return true;
    }

    for (const char* symbol = text; *symbol != '\0'; ++symbol)
    {
        if (*symbol != ' ' && *symbol != '\t')
        {
            return false;
        }
    }
    return true;
}

bool Task::IsValidCategory(TaskCategory category)
{
    switch (category)
    {
    case TaskCategory::BUG:
    case TaskCategory::FEATURE:
    case TaskCategory::SUPPORT:
        return true;
    default:
        return false;
    }
}

void Task::CheckHours(const double& hours)
{
    if (hours <= 0.0)
    {
        throw InvalidHoursException();
    }
    if (hours > MAX_HOURS)
    {
        throw InvalidHoursException();
    }
}

const char* Task::CategoryToText(TaskCategory category)
{
    switch (category)
    {
    case TaskCategory::BUG:
        return "Bug";
    case TaskCategory::FEATURE:
        return "New feature";
    case TaskCategory::SUPPORT:
        return "Support";
    default:
        return "Unknown";
    }
}

// ---------- Static public ----------

time_t Task::MakeDate(
    int day,
    int month,
    int year)
{
    tm timeInfo = {};
    timeInfo.tm_mday = day;
    timeInfo.tm_mon = month - 1;
    timeInfo.tm_year = year - TM_YEAR_BASE;
    timeInfo.tm_hour = END_OF_DAY_HOUR;
    timeInfo.tm_min = END_OF_DAY_MINUTE;
    timeInfo.tm_sec = END_OF_DAY_SECOND;
    timeInfo.tm_isdst = AUTO_DAYLIGHT_SAVING;

    time_t result = mktime(&timeInfo);
    if (result == INVALID_TIME)
    {
        throw InvalidDateException();
    }

    // mktime normalizes dates like 31.02 into 03.03, so compare fields back.
    bool isSameDay = (timeInfo.tm_mday == day);
    bool isSameMonth = (timeInfo.tm_mon == month - 1);
    bool isSameYear = (timeInfo.tm_year == year - TM_YEAR_BASE);
    if (!isSameDay || !isSameMonth || !isSameYear)
    {
        throw InvalidDateException();
    }
    return result;
}

// ---------- Constructors / destructor ----------

Task::Task(
    const char* const name,
    TaskCategory category,
    const time_t& deadline)
    : m_name(nullptr)
    , m_deadline(deadline)
    , m_category(category)
    , m_isClosed(false)
    , m_assignee(nullptr)
    , m_estimatedHours(0.0)
    , m_spentHours(0.0)
{
    if (IsBlank(name))
    {
        throw EmptyNameException();
    }
    if (!IsValidCategory(category))
    {
        throw InvalidCategoryException();
    }
    if (deadline <= 0)
    {
        throw InvalidDateException();
    }

    // Allocate only after all checks, so nothing leaks on exception.
    m_name = CopyString(name);
}

Task::Task(const Task& other)
    : m_name(CopyString(other.m_name))
    , m_deadline(other.m_deadline)
    , m_category(other.m_category)
    , m_isClosed(other.m_isClosed)
    , m_assignee(nullptr)
    , m_estimatedHours(other.m_estimatedHours)
    , m_spentHours(other.m_spentHours)
{
    if (other.m_assignee != nullptr)
    {
        m_assignee = CopyString(other.m_assignee);
    }
}

Task& Task::operator=(const Task& other)
{
    if (this == &other)
    {
        return *this;
    }

    char* newName = CopyString(other.m_name);
    char* newAssignee = nullptr;
    if (other.m_assignee != nullptr)
    {
        newAssignee = CopyString(other.m_assignee);
    }

    delete[] m_name;
    delete[] m_assignee;

    m_name = newName;
    m_assignee = newAssignee;
    m_deadline = other.m_deadline;
    m_category = other.m_category;
    m_isClosed = other.m_isClosed;
    m_estimatedHours = other.m_estimatedHours;
    m_spentHours = other.m_spentHours;
    return *this;
}

Task::~Task()
{
    delete[] m_name;
    delete[] m_assignee;
}

// ---------- Selectors ----------

const char* Task::GetName() const
{
    return m_name;
}

time_t Task::GetDeadline() const
{
    return m_deadline;
}

TaskCategory Task::GetCategory() const
{
    return m_category;
}

bool Task::IsClosed() const
{
    return m_isClosed;
}

const char* Task::GetAssignee() const
{
    if (m_assignee == nullptr)
    {
        return NOT_ASSIGNED_TEXT;
    }
    return m_assignee;
}

double Task::GetEstimatedHours() const
{
    return m_estimatedHours;
}

double Task::GetSpentHours() const
{
    return m_spentHours;
}

// ---------- Modifiers ----------

void Task::SetAssignee(const char* const assignee)
{
    if (IsBlank(assignee))
    {
        throw EmptyAssigneeException();
    }
    if (m_isClosed)
    {
        throw TaskClosedException();
    }

    char* newAssignee = CopyString(assignee);
    delete[] m_assignee;
    m_assignee = newAssignee;
}

void Task::SetEstimate(const double& hours)
{
    CheckHours(hours);
    if (m_isClosed)
    {
        throw TaskClosedException();
    }
    m_estimatedHours = hours;
}

void Task::AddSpentTime(const double& hours)
{
    CheckHours(hours);
    if (m_isClosed)
    {
        throw TaskClosedException();
    }
    m_spentHours += hours;
}

void Task::Close()
{
    if (m_isClosed)
    {
        throw TaskClosedException();
    }
    m_isClosed = true;
}

// ---------- Queries ----------

bool Task::IsOverdue() const
{
    if (m_isClosed)
    {
        return false;
    }

    time_t now = time(nullptr);
    return now > m_deadline;
}

void Task::Print() const
{
    char dateText[DATE_BUFFER_SIZE];
    tm timeInfo;
#ifdef _WIN32
    localtime_s(&timeInfo, &m_deadline);
#else
    localtime_r(&m_deadline, &timeInfo);
#endif
    strftime(dateText, DATE_BUFFER_SIZE, DATE_FORMAT, &timeInfo);

    const char* statusText = "open";
    if (m_isClosed)
    {
        statusText = "closed";
    }

    const char* overdueText = "no";
    if (IsOverdue())
    {
        overdueText = "yes";
    }

    std::cout << "-----------------------------\n";
    std::cout << "Task:      " << m_name << '\n';
    std::cout << "Category:  " << CategoryToText(m_category) << '\n';
    std::cout << "Deadline:  " << dateText << '\n';
    std::cout << "Status:    " << statusText << '\n';
    std::cout << "Assignee:  " << GetAssignee() << '\n';
    std::cout << "Estimate:  " << m_estimatedHours << " h\n";
    std::cout << "Spent:     " << m_spentHours << " h\n";
    std::cout << "Overdue:   " << overdueText << '\n';
}
