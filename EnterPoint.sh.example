#!/bin/bash

VERSION=0.3.0

Image="do.17bdc.com/shanbay/farmer:${VERSION}-runtime"
TestImage="do.17bdc.com/shanbay/farmer:${VERSION}-test"
AirflowImage="do.17bdc.com/shanbay/farmer:${VERSION}-airflow"

CurDir="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )"
LogDir="/data/farmer/logs"
TmpDir="/data/farmer/tmp"


if [[ -a ${CurDir}/envs.sh ]]; then
    source ${CurDir}/envs.sh
fi


build-img() {
    version=$1
    shift;

    for tag in runtime test airflow; do
        docker build -t farmer:${tag} -f \
               ${CurDir}/dockerfiles/Dockerfile-${tag} \
               ${CurDir}/dockerfiles
        docker tag farmer:${tag} \
               do.17bdc.com/shanbay/farmer:${version}-${tag}
        if [[ "$1" == "-u" ]]; then
            docker push do.17bdc.com/shanbay/farmer:${version}-${tag}
        fi
    done
}


create_data_volume() {
    docker inspect farmer-data &> /dev/null
    if [[ "$?" == "1" ]]; then
        docker create --name farmer-data \
               -v ${CurDir}:/usr/src/app \
               -v ${LogDir}:/usr/src/app/logs \
               -v ${TmpDir}:/usr/src/app/tmp \
               do.17bdc.com/alpine:3.3 /bin/true

        docker run --rm --volumes-from farmer-data \
               ${Image} mkdir -p logs/cron/
    fi
}


run() {
    create_data_volume

    docker run --rm --net=host \
           -e "NodeEnv=${NodeEnv}" \
           -e "TZ=Asia/Shanghai" \
           -e "PYTHONPATH=/usr/src/app/" \
           --volumes-from farmer-data \
           ${Image} \
           python main.py "$@"
}


start() {
    create_data_volume

    docker run -d --net=host \
           --name farmer \
           -e "NodeEnv=${NodeEnv}" \
           -e "TZ=Asia/Shanghai" \
           -e "PYTHONPATH=/usr/src/app/" \
           --volumes-from farmer-data \
           ${Image} \
           python main.py flow "$@"
}


stop() {
    docker stop farmer 2>/dev/null
    docker rm -v farmer 2>/dev/null
    docker rm -v farmer-data 2>/dev/null
}


shell() {
    create_data_volume

    docker run -it --rm --net=host \
           -e "TZ=Asia/Shanghai" \
           -e "NodeEnv=${NodeEnv}" \
           -e "PYTHONPATH=/usr/src/app/" \
           --volumes-from farmer-data \
           ${Image} \
           zsh
}


test() {
    create_data_volume

    docker run -it --rm --net=host \
           -e "TZ=Asia/Shanghai" \
           -e "NodeEnv=test" \
           -e "PYTHONPATH=/usr/src/app/" \
           --volumes-from farmer-data \
           ${TestImage} \
           python -m unittest "$@"
}


################################


airflow() {
    action="$1"
    shift

    case "$action" in
        master) master "$@";;
        worker) worker "$@";;
        *)
            echo "Usage: "
            echo "./farmer.sh airflow master"
            echo "./farmer.sh airflow worker"
            exit 1
            ;;
    esac
}


master() {
    stop-master() {
        echo "... Stopping ..."
        for name in webserver flower scheduler; do
            docker inspect ${name} &> /dev/null
            if [[ "$?" == "0" ]]; then
                docker stop ${name}
                docker rm ${name}
            fi
        done
    }

    start-master() {
        echo "... Running ..."
        docker run -d  \
               --name webserver \
               -e "NodeEnv=${NodeEnv}" \
               -e "TZ=Asia/Shanghai" \
               -e "PYTHONPATH=/usr/src/app/" \
               -e "LOAD_EX=n" \
               -e "FERNET_KEY=${FERNET_KEY}" \
               -e "EXECUTOR=Celery" \
               -e "REDIS_PASSWORD=${REDIS_PASSWORD}" \
               -e "REDIS_HOST=${REDIS_HOST}" \
               -e "REDIS_PORT=${REDIS_PORT}" \
               -e "POSTGRES_USER=${POSTGRES_USER}" \
               -e "POSTGRES_PASSWORD=${POSTGRES_PASSWORD}" \
               -e "POSTGRES_DB=${POSTGRES_DB}" \
               -e "POSTGRES_HOST=${POSTGRES_HOST}" \
               -p 8080:8080 \
               --restart=always \
               -v ${CurDir}/dags:/usr/local/airflow/dags \
               ${AirflowImage} \
               webserver

        docker run -d \
               --name flower \
               -e "NodeEnv=${NodeEnv}" \
               -e "TZ=Asia/Shanghai" \
               -e "PYTHONPATH=/usr/src/app/" \
               -e "EXECUTOR=Celery" \
               -e "REDIS_PASSWORD=${REDIS_PASSWORD}" \
               -e "REDIS_HOST=${REDIS_HOST}" \
               -e "REDIS_PORT=${REDIS_PORT}" \
               -p 5555:5555 \
               --restart=always \
               ${AirflowImage} \
               flower

        docker run -d  \
               --name scheduler \
               -e "NodeEnv=${NodeEnv}" \
               -e "TZ=Asia/Shanghai" \
               -e "PYTHONPATH=/usr/src/app/" \
               -e "LOAD_EX=n" \
               -e "FERNET_KEY=${FERNET_KEY}" \
               -e "EXECUTOR=Celery" \
               -e "REDIS_PASSWORD=${REDIS_PASSWORD}" \
               -e "POSTGRES_USER=${POSTGRES_USER}" \
               -e "POSTGRES_PASSWORD=${POSTGRES_PASSWORD}" \
               -e "POSTGRES_DB=${POSTGRES_DB}" \
               -e "POSTGRES_HOST=${POSTGRES_HOST}" \
               -e "REDIS_HOST=${REDIS_HOST}" \
               -e "REDIS_PORT=${REDIS_PORT}" \
               --restart=always \
               -v ${CurDir}/dags:/usr/local/airflow/dags \
               ${AirflowImage} \
               scheduler
    }

    action="$1"
    case "$action" in
        start) start-master;;
        stop) stop-master;;
        restart)
            stop-master
            start-master ;;
        *)
            echo "Usage: "
            echo "./farmer.sh airflow master start"
            echo "./farmer.sh airflow master stop"
            echo "./farmer.sh airflow master restart"
            exit 1
            ;;
    esac
}


worker() {
    stop-worker(){
        docker inspect worker &> /dev/null
        if [[ "$?" == "0" ]]; then
            echo "... Stopping ..."
            docker stop worker
            docker rm worker
        fi
    }

    start-worker(){
        echo "... Running ..."
        docker run -d \
               --name worker \
               -e "NodeEnv=${NodeEnv}" \
               -e "TZ=Asia/Shanghai" \
               -e "PYTHONPATH=/usr/src/app/" \
               -e "FERNET_KEY=${FERNET_KEY}" \
               -e "EXECUTOR=Celery" \
               -e "REDIS_PASSWORD=${REDIS_PASSWORD}" \
               -e "REDIS_HOST=${REDIS_HOST}" \
               -e "REDIS_PORT=${REDIS_PORT}" \
               -e "POSTGRES_HOST=${POSTGRES_HOST}" \
               -e "POSTGRES_PORT=${POSTGRES_PORT}" \
               -e "POSTGRES_USER=${POSTGRES_USER}" \
               -e "POSTGRES_PASSWORD=${POSTGRES_PASSWORD}" \
               -e "POSTGRES_DB=${POSTGRES_DB}" \
               -v ${CurDir}/dags:/usr/local/airflow/dags \
               --restart=always \
               ${AirflowImage} \
               worker
    }

    action="$1"
    case "$action" in
        start) start-worker;;
        stop) stop-worker;;
        restart)
            stop-worker
            start-worker ;;
        *)
            echo "Usage: "
            echo "./farmer.sh airflow worker start"
            echo "./farmer.sh airflow worker stop"
            echo "./farmer.sh airflow worker restart"
            exit 1
            ;;
    esac
}


################################
#       Start of Script        #
################################


Action=$1

shift
case "$Action" in
    build-img) build-img "$@";;
    run) run "$@" ;;
    shell) shell "$@" ;;
    start) start "$@" ;;
    stop) stop "$@" ;;
    airflow) airflow "$@" ;;
    restart)
        stop
        start
        ;;
    test) test "$@" ;;
    *)
        echo "Usage:"
        echo "./farmer.sh build-img version [-u]"
        echo "./farmer.sh run"
        echo "./farmer.sh shell"
        echo "./farmer.sh test"
        echo "./farmer.sh start | stop | restart"
        echo "./farmer.sh airflow"
        exit 1
        ;;
esac

exit 0
