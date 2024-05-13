JDBC常用异常：
    查询为空：NullPointerException
    数据访问异常：DataAccessException

    无法捕捉SQLIntegrityConstraintViolationException：
        SQLIntegrityConstraintViolationException的祖宗们：
            SQLNonTransientException，SQLException，Exception